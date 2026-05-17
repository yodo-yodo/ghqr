# FGPAT compatibility and ghqr impact

この資料は、GitHub personal access token (classic) と Fine-grained personal access token (FGPAT) の差分、既存の限定実測結果、ghqr への影響、README / warning 改善案を 1 つに統合した根拠資料である。

推論で断定しない。公式 Docs で確認できない項目は `未確認` と書く。実 organization、repository、enterprise、username、token 値、token の一部は記録しない。

## 参照資料

公式 Docs:

- GitHub Docs: Endpoints available for fine-grained personal access tokens
  - https://docs.github.com/en/rest/authentication/endpoints-available-for-fine-grained-personal-access-tokens
- GitHub Docs: Permissions required for fine-grained personal access tokens
  - https://docs.github.com/en/rest/authentication/permissions-required-for-fine-grained-personal-access-tokens
- GitHub Docs: Managing your personal access tokens
  - https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens
- GitHub Docs: Setting a personal access token policy for your organization
  - https://docs.github.com/en/organizations/managing-programmatic-access-to-your-organization/setting-a-personal-access-token-policy-for-your-organization
- GitHub Docs: Managing requests for personal access tokens in your organization
  - https://docs.github.com/en/organizations/managing-programmatic-access-to-your-organization/managing-requests-for-personal-access-tokens-in-your-organization
- GitHub Docs: Forming calls with GraphQL
  - https://docs.github.com/en/graphql/guides/forming-calls-with-graphql
- GitHub Docs: REST API endpoints for enterprise audit logs
  - https://docs.github.com/en/enterprise-cloud@latest/rest/enterprise-admin/audit-log
- GitHub Docs: REST API endpoints for Copilot user management
  - https://docs.github.com/en/rest/copilot/copilot-user-management
- GitHub Docs: REST API endpoints for code security configurations
  - https://docs.github.com/en/rest/code-security/configurations

補助資料:

- GitHub Blog: Introducing fine-grained personal access tokens
  - https://github.blog/security/application-security/introducing-fine-grained-personal-access-tokens-for-github/

ghqr ソースコード:

- `internal/config/github.go`
- `internal/pipeline/stage_initialization.go`
- `internal/scanners/graphql_client.go`
- `internal/scanners/batch.go`
- `internal/scanners/ruleset.go`
- `internal/scanners/organization.go`
- `internal/scanners/enterprise.go`

## 1. 結論

- FGPAT は Classic PAT の完全上位互換ではない。
  - 根拠: GitHub Docs: Managing your personal access tokens は、FGPAT には Resource owner、Repository access、Repository permissions、Organization permissions、Account permissions があることを説明している。
  - 根拠: 同 Docs は、outside collaborators が organization repositories へアクセスする場合は personal access tokens (classic) のみ使えると説明している。
- FGPAT は Resource owner / repository access / permission / organization approval の制約を受ける。
  - 根拠: GitHub Docs: Managing your personal access tokens
  - 根拠: GitHub Docs: Setting a personal access token policy for your organization
  - 根拠: GitHub Docs: Managing requests for personal access tokens in your organization
- REST API は、FGPAT 対応 endpoint が公式 Docs で列挙されている。
  - 根拠: GitHub Docs: Endpoints available for fine-grained personal access tokens
  - 根拠: GitHub Docs: Permissions required for fine-grained personal access tokens
- Endpoint ごとに必要 permission が異なる。
  - 根拠: GitHub Docs: Permissions required for fine-grained personal access tokens
- Classic PAT のみと公式 Docs で確認できた操作がある。
  - outside collaborator による organization repository access
  - 自分または自分が member である organization が owner ではない public repository への write access
  - Enterprise audit log streaming configuration 系 endpoint
  - Enterprise code security configuration endpoint の一部
- FGPAT 固有または FGPAT で明確に細かく制御できるものがある。
  - Resource owner
  - Repository access の限定
  - Repository / Organization / Account permission の細分化
  - Organization owner による FGPAT request approval / deny
- ghqr への影響:
  - Repository 明示指定 scan は、既存の限定実測で Classic PAT と FGPAT の両方で成功済み。
  - Organization / Enterprise / audit / Copilot / security endpoint は、FGPAT の Resource owner、Repository access、permission、organization approval、enterprise role、GitHub product availability に影響される。
  - ghqr ソースコード上、token 種別判定はしていない。`GH_TOKEN`、次に `GITHUB_TOKEN` を読み、同じ token を REST / GraphQL client へ渡す。
  - 現時点の既存実測では、REST / GraphQL 単体では成功するが ghqr だけ失敗する箇所は確認されていない。

## 1.1 ghqr source code inventory

ghqr は token 種別で分岐しない。`GH_TOKEN` を優先し、空なら `GITHUB_TOKEN` を読み、同じ token transport を REST / GraphQL / raw HTTP client で共有する。

| 領域 | ファイル | 確認内容 |
|---|---|---|
| token / client initialization | `internal/config/github.go`, `internal/pipeline/stage_initialization.go` | `GH_TOKEN`、次に `GITHUB_TOKEN` を読む。`oauth2.StaticTokenSource` で REST / GraphQL client を作る。認証確認は REST `GET /user` 相当。 |
| repository scan | `internal/scanners/graphql_client.go`, `internal/scanners/batch.go`, `internal/scanners/ruleset.go` | GraphQL `repository(owner:, name:)`、repository metadata、branch protection、rulesets、vulnerability alerts、collaborators、deploy keys、file presence checks を使う。 |
| organization scan / discovery | `internal/scanners/organization.go`, `internal/pipeline/stage_org_repository_scan.go` | REST org endpoint と GraphQL `organization.repositories` を使う。Actions、Dependabot、code scanning、secret scanning、security managers、Copilot billing を endpoint 単位で取得する。 |
| enterprise scan / discovery | `internal/scanners/enterprise.go`, `internal/pipeline/stage_enterprise_discovery.go` | GraphQL `viewer.enterprises`、`enterprise(slug:)`、`enterprise.organizations` と REST enterprise audit / security / code security endpoint を使う。 |
| report rendering | `internal/renderers` | JSON / Markdown / Excel rendering は token 種別に依存しない。 |

## 2. 用語定義

| 用語 | 定義 | 根拠URL |
|---|---|---|
| Classic PAT | GitHub の personal access token (classic)。Scope ベースで権限を付与する token。 | https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens |
| Fine-grained PAT | Resource owner、Repository access、Repository / Organization / Account permissions を設定する personal access token。 | https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens |
| Resource owner | FGPAT がアクセスする user または organization owner。Organization resource へアクセスする FGPAT は対象 organization を Resource owner にする必要がある。 | https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens |
| Repository access | FGPAT がアクセスできる repository 範囲。All repositories または Only select repositories を選ぶ。 | https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens |
| Repository permissions | FGPAT の repository 単位 permission。Contents、Metadata、Administration などがある。 | https://docs.github.com/en/rest/authentication/permissions-required-for-fine-grained-personal-access-tokens |
| Organization permissions | FGPAT の organization 単位 permission。Administration、Members、GitHub Copilot Business などが endpoint ごとに要求される。 | https://docs.github.com/en/rest/authentication/permissions-required-for-fine-grained-personal-access-tokens |
| Account permissions | FGPAT の account 単位 permission。User 関連 endpoint に対応する permission 群。 | https://docs.github.com/en/rest/authentication/permissions-required-for-fine-grained-personal-access-tokens |
| Organization approval | Organization が approval を要求する場合、organization owner が FGPAT request を approve するまで private resource へ使えない制御。 | https://docs.github.com/en/organizations/managing-programmatic-access-to-your-organization/managing-requests-for-personal-access-tokens-in-your-organization |
| Enterprise access | Enterprise admin role や enterprise permission / Classic PAT scope が必要な enterprise-level API access。 | https://docs.github.com/en/enterprise-cloud@latest/rest/enterprise-admin/audit-log |
| REST endpoint | GitHub REST API の method + path 単位の API。FGPAT 対応可否と permission は endpoint ごとに公式 Docs で確認する。 | https://docs.github.com/en/rest/authentication/endpoints-available-for-fine-grained-personal-access-tokens |
| GraphQL field/query | GitHub GraphQL API の query / field。GraphQL docs は、要求する data によって必要 scope / permission が決まると説明している。 | https://docs.github.com/en/graphql/guides/forming-calls-with-graphql |
| Org policy / PAT access | Organization owner が personal access token の access policy を設定し、Classic PAT または FGPAT の organization resource access を制御する仕組み。 | https://docs.github.com/en/organizations/managing-programmatic-access-to-your-organization/setting-a-personal-access-token-policy-for-your-organization |
| Access 対象外 repository | FGPAT の Repository access に含まれていない repository。Private repository では repository が存在しないように見える結果になることがある。 | https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens |
| Archived repository | Archived 状態の repository。ghqr 検証では private / public repository とは別に、読み取り結果と endpoint 挙動を分けて記録する対象。 | 未確認 |
| Disabled repository | 利用停止または無効化された repository。GitHub Docs 上の FGPAT 固有差分はこの資料では未確認。 | 未確認 |
| Transferred repository | owner が移管された repository。FGPAT の Resource owner / Repository access と一致しない場合は別 owner の resource として扱う必要がある。 | https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens |
| Organization discovery | ghqr が user から見える organization を自動列挙する処理。REST `GET /user/orgs` と GraphQL organization query の可視性に依存する。 | https://docs.github.com/en/rest/authentication/endpoints-available-for-fine-grained-personal-access-tokens |
| Organization repositories GraphQL | ghqr が GraphQL `organization(login:) { repositories(...) }` で organization 配下 repository を列挙する処理。Organization 全体 scan では API volume が増える。 | https://docs.github.com/en/graphql/guides/forming-calls-with-graphql |
| Rate limit | GitHub API の rate limit。ghqr は HTTP transport で rate limit response を扱うが、検証では API volume を最小化する。 | https://docs.github.com/en/rest/using-the-rest-api/rate-limits-for-the-rest-api |
| GraphQL cost limit | GitHub GraphQL API の query cost / resource limit。ghqr は org-wide batch scan で一部 connection を省略し、batch size を小さくしている。 | https://docs.github.com/en/graphql/overview/rate-limits-and-query-limits-for-the-graphql-api |

## 3. Classic PAT のみ可能なもの

公式 Docs で Classic PAT のみ、または FGPAT ではできないと確認できたものだけを記録する。

| 区分 | 操作/API | REST/GraphQL/UI | Classic PAT | FGPAT | 根拠URL | 原文要約 | ghqr影響 |
|---|---|---|---|---|---|---|---|
| Repository access | Outside collaborator が collaborator である organization repository へアクセスする | Token behavior | 可能 | 非対応 | https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens | Outside collaborators can only use personal access tokens (classic) to access organization repositories that they are a collaborator on. | FGPAT で outside collaborator 前提の org repo scan は README で制約説明が必要。 |
| Repository write access | 自分または member organization が owner ではない public repository への write access | Token behavior | 可能 | 非対応 | https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens | Only personal access tokens (classic) have write access for public repositories not owned by you or an organization you are not a member of. | ghqr は read scan が主目的のため直接影響は限定的。 |
| Enterprise audit log streaming | `GET /enterprises/{enterprise}/audit-log/stream-key` | REST | 対応。endpoint page は audit log endpoints が personal access token (classic) 認証のみを support すると説明している。個別 scope は未確認。 | 非対応 | https://docs.github.com/en/enterprise-cloud@latest/rest/enterprise-admin/audit-log | This endpoint does not work with GitHub App user access tokens, GitHub App installation access tokens, or fine-grained personal access tokens. | ghqr は現時点で stream-key を使わない。 |
| Enterprise audit log streaming | `GET /enterprises/{enterprise}/audit-log/streams` | REST | 対応。endpoint page は audit log endpoints が personal access token (classic) 認証のみを support すると説明している。個別 scope は未確認。 | 非対応 | https://docs.github.com/en/enterprise-cloud@latest/rest/enterprise-admin/audit-log | This endpoint does not work with GitHub App user access tokens, GitHub App installation access tokens, or fine-grained personal access tokens. | ghqr は現時点で audit log stream configuration を使わない。 |
| Enterprise code security configurations | `GET /enterprises/{enterprise}/code-security/configurations` | REST | Enterprise admin + `read:enterprise` scope が必要 | 非対応 | https://docs.github.com/en/rest/code-security/configurations | This endpoint does not work with GitHub App user access tokens, GitHub App installation access tokens, or fine-grained personal access tokens. | ghqr の `GET /enterprises/{enterprise}/code_security/settings` とは path が異なる。関連機能として enterprise GHAS 設定検証時に注意が必要。 |
| Enterprise code security configurations | `POST /enterprises/{enterprise}/code-security/configurations` | REST | Enterprise admin + `admin:enterprise` scope が必要 | 非対応 | https://docs.github.com/en/rest/code-security/configurations | This endpoint does not work with GitHub App user access tokens, GitHub App installation access tokens, or fine-grained personal access tokens. | ghqr は write endpoint を使わない。 |

確認候補の扱い:

| 確認候補 | 状態 | 根拠 | ghqr影響 |
|---|---|---|---|
| `GET /enterprises/{enterprise}/audit-log` | FGPAT 非対応として扱う。Docs の冒頭は personal access token (classic) のみを明記し、Fine-grained access tokens section は GitHub App token types のみを列挙している。 | https://docs.github.com/en/enterprise-cloud@latest/rest/enterprise-admin/audit-log | ghqr enterprise audit log scan は Classic PAT でも enterprise admin + scope が必要。FGPAT では GitHub App token とは別扱いとして README 注意が必要。 |
| Organization PAT 管理系 endpoint | 未確認 | この資料では ghqr が直接使う endpoint を優先した。 | ghqr は現時点で PAT 管理 endpoint を使わない。 |
| GraphQL enterprise fields | 未確認 | GraphQL docs は field 単位 FGPAT 差分をこの資料で確認できる形では列挙していない。 | `viewer.enterprises`、`enterprise(slug:)`、`enterprise.organizations` は実測で権限不足 warning を確認済み。 |
| GHAS settings | 一部 Classic PAT only endpoint を確認。ghqr の `code_security/settings` path は公式 Docs 上の該当確認が未完了。 | https://docs.github.com/en/rest/code-security/configurations | ghqr enterprise GHAS 設定は `403` / `404` を許容する現実的な warning が必要。 |

## 4. FGPAT のみ、または FGPAT が明確に優位なもの

| 区分 | 機能 | Classic PAT | FGPAT | 根拠URL | 原文要約 | ghqr影響 |
|---|---|---|---|---|---|---|
| Resource scoping | Resource owner を選べる | scope ベース。Resource owner 選択という UI 制御は未確認 | User または organization を Resource owner として選択 | https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens | FGPAT 作成時に Resource owner を選択する。 | org private repo scan では Resource owner 誤りが `404` の原因になる。 |
| Repository scoping | Repository access を限定できる | repo scope は repository 単位選択ではない | All repositories または Only select repositories | https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens | FGPAT で repository access を選択する。 | 初期検証は Only select repositories + repository 明示指定が安全。 |
| Permission granularity | Permission を endpoint 群ごとに細分化できる | scope ベース | Repository / Organization / Account permission を個別設定 | https://docs.github.com/en/rest/authentication/permissions-required-for-fine-grained-personal-access-tokens | Endpoint ごとの required permission が列挙されている。 | ghqr は機能ごとに必要 permission が異なる。README に最小 permission と追加 permission を分けて書く。 |
| Org approval | FGPAT request approval / deny | approval policy の対象外 | Organization owner が approve / deny できる | https://docs.github.com/en/organizations/managing-programmatic-access-to-your-organization/managing-requests-for-personal-access-tokens-in-your-organization | Organization owner can approve or deny FGPAT requests. | `403` / `404` 時に org approval 未完了を troubleshooting に含める。 |
| Lifetime policy | FGPAT 最大 lifetime の default policy | Personal access tokens (classic) do not have an expiration requirement と Docs が説明 | Organization の FGPAT default maximum lifetime は 366 days | https://docs.github.com/en/organizations/managing-programmatic-access-to-your-organization/setting-a-personal-access-token-policy-for-your-organization | Organization owners can set maximum lifetime policies; FGPAT default maximum lifetime policy is 366 days. | 長期運用の README で token rotation / expiration を注意書きする。 |

## 5. REST API endpoint 差分

ghqr が利用する REST endpoint と、prompt で確認対象に指定された REST endpoint を中心に整理する。FGPAT 対応は `Endpoints available for fine-grained personal access tokens` と endpoint 個別 Docs で確認した範囲に限定する。Classic PAT scope は公式 Docs でこの資料内で確認できたものだけ記録し、それ以外は `未確認` とする。

| ghqr機能 | endpoint | FGPAT対応 | 必要permission | Classic PAT scope | 根拠URL | 実測状況 | 備考 |
|---|---|---|---|---|---|---|---|
| authentication | `GET /user` | 対応 | User account read 系。詳細 permission は未確認 | 未確認 | https://docs.github.com/en/rest/authentication/endpoints-available-for-fine-grained-personal-access-tokens | Classic PAT / FGPAT とも authentication 成功済み | ghqr initialization が `Users.Get(ctx, "")` で使用。 |
| organization discovery | `GET /user/orgs` | 対応 | User / organization visibility に依存。詳細 permission は未確認 | 未確認 | https://docs.github.com/en/rest/authentication/endpoints-available-for-fine-grained-personal-access-tokens | 既存実測で organization auto-discovery 成功 | ghqr auto-discovery で使用。 |
| organization scan | `GET /orgs/{org}` | 対応 | Organization metadata visibility と approval に依存。詳細 permission は未確認 | 未確認 | https://docs.github.com/en/rest/authentication/endpoints-available-for-fine-grained-personal-access-tokens | Classic PAT / FGPAT とも scan 内で成功 | ghqr `OrganizationScanner.ScanAll` で使用。 |
| organization scan | `GET /orgs/{org}/repos` | 対応 | Repository access / organization visibility に依存。詳細 permission は未確認 | 未確認 | https://docs.github.com/en/rest/authentication/endpoints-available-for-fine-grained-personal-access-tokens | 既存実測では GraphQL repository enumeration を主に使用 | Organization 全体 scan は初期検証禁止。 |
| repository scan | `GET /repos/{owner}/{repo}` | 対応 | Metadata read。Private repo は Resource owner / Repository access / approval が必要 | 未確認 | https://docs.github.com/en/rest/authentication/endpoints-available-for-fine-grained-personal-access-tokens | Classic PAT / FGPAT とも REST 単体成功済み | Explicit repository scan の事前 visibility check に使える。 |
| organization actions | `GET /orgs/{org}/actions/permissions` | 対応 | Actions / Administration 系 organization permission。詳細は endpoint Docs 参照 | 未確認 | https://docs.github.com/en/rest/authentication/endpoints-available-for-fine-grained-personal-access-tokens | Classic PAT / FGPAT とも既存実測で `403` | token 種別差ではなく権限不足として記録済み。 |
| organization actions | `GET /orgs/{org}/actions/permissions/workflow` | 対応 | Actions / Administration 系 organization permission。詳細は endpoint Docs 参照 | 未確認 | https://docs.github.com/en/rest/authentication/endpoints-available-for-fine-grained-personal-access-tokens | 未確認 | ghqr は actions permissions 取得後に workflow permissions も取得する。 |
| org security alerts | `GET /orgs/{org}/dependabot/alerts` | 対応 | Dependabot alerts read 系。詳細 permission は endpoint Docs 参照 | 未確認 | https://docs.github.com/en/rest/authentication/endpoints-available-for-fine-grained-personal-access-tokens | Classic PAT / FGPAT とも REST 単体成功済み | GHAS / Dependabot availability に依存。 |
| org security alerts | `GET /orgs/{org}/code-scanning/alerts` | 対応 | Code scanning alerts read 系。詳細 permission は endpoint Docs 参照 | 未確認 | https://docs.github.com/en/rest/authentication/endpoints-available-for-fine-grained-personal-access-tokens | Classic PAT / FGPAT とも REST 単体成功済み | GHAS / code scanning availability に依存。 |
| org security alerts | `GET /orgs/{org}/secret-scanning/alerts` | 対応 | Secret scanning alerts read 系。詳細 permission は endpoint Docs 参照 | 未確認 | https://docs.github.com/en/rest/authentication/endpoints-available-for-fine-grained-personal-access-tokens | Classic PAT / FGPAT とも REST 単体成功済み | GHAS / secret scanning availability に依存。 |
| org security manager | `GET /orgs/{org}/security-managers` | 対応 | Organization security manager visibility。詳細 permission は未確認 | 未確認 | https://docs.github.com/en/rest/authentication/endpoints-available-for-fine-grained-personal-access-tokens | Classic PAT / FGPAT とも REST 単体成功済み | `403` / `404` は ghqr で empty result 扱い。 |
| EMU fallback | `GET /orgs/{org}/external-groups` | 対応 | EMU organization / external groups access。詳細 permission は未確認 | 未確認 | https://docs.github.com/en/rest/authentication/endpoints-available-for-fine-grained-personal-access-tokens | 未確認 | ghqr は GraphQL enterprise IdP 失敗時の fallback として使用。 |
| Copilot | `GET /orgs/{org}/copilot/billing` | 対応 | `GitHub Copilot Business` organization permission read または `Administration` organization permission read | `manage_billing:copilot` または `read:org` | https://docs.github.com/en/rest/copilot/copilot-user-management | Classic PAT / FGPAT とも REST 単体成功済み | Organization owner だけが view 可能と Docs が説明。 |
| Enterprise audit | `GET /enterprises/{enterprise}/audit-log` | FGPAT 非対応として扱う。Fine-grained access token types に FGPAT が含まれない | GitHub App user / installation token は Enterprise administration read。FGPAT は未対応 | `read:audit_log` | https://docs.github.com/en/enterprise-cloud@latest/rest/enterprise-admin/audit-log | Classic PAT / FGPAT とも既存実測で `403` | Classic PAT でも enterprise admin + scope が必要。ghqr は warning 継続。 |
| Enterprise security alerts | `GET /enterprises/{enterprise}/dependabot/alerts` | 対応リスト上は未確認 | 未確認 | 未確認 | https://docs.github.com/en/rest/authentication/endpoints-available-for-fine-grained-personal-access-tokens | 未確認 | ghqr enterprise scan で使用。Enterprise 全体 scan は初期検証禁止。 |
| Enterprise security alerts | `GET /enterprises/{enterprise}/code-scanning/alerts` | 対応リスト上は未確認 | 未確認 | 未確認 | https://docs.github.com/en/rest/authentication/endpoints-available-for-fine-grained-personal-access-tokens | 未確認 | ghqr enterprise scan で使用。 |
| Enterprise security alerts | `GET /enterprises/{enterprise}/secret-scanning/alerts` | 対応リスト上は未確認 | 未確認 | 未確認 | https://docs.github.com/en/rest/authentication/endpoints-available-for-fine-grained-personal-access-tokens | 未確認 | ghqr enterprise scan で使用。 |
| Enterprise GHAS settings | `GET /enterprises/{enterprise}/code_security/settings` | 未確認 | 未確認 | 未確認 | https://docs.github.com/en/rest/code-security/configurations | 既存実測で `403` | GitHub Docs で確認できた `code-security/configurations` endpoint とは path が異なるため転用しない。 |

## 5.1 FGPAT で詰まりやすい候補

この表は「FGPAT では常に失敗する」という意味ではない。公式 Docs と既存実測から、FGPAT 設定、GitHub permission、GitHub product availability、enterprise / organization policy により差分が出る領域を整理する。

| ghqrコマンド/機能 | FGPATで詰まる条件 | 一次分類 | 根拠 | ghqrでの扱い |
|---|---|---|---|---|
| `ghqr scan --repository ORG/REPO` | Resource owner が対象 organization ではない | FGPAT設定不足 | この資料の既存実測サマリ | REST `GET /repos/ORG/REPO` は `404`、GraphQL repository resolve も失敗した既存実測あり。 |
| `ghqr scan --repository ORG/REPO` | Repository access に対象 repository が含まれていない | FGPAT設定不足 | https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens | `404` または repository resolve failure として見えるため、troubleshooting で確認対象にする。 |
| `ghqr scan --repository ORG/REPO` | `Contents: Read-only` がない | FGPAT permission不足 | https://docs.github.com/en/rest/authentication/permissions-required-for-fine-grained-personal-access-tokens | file presence checks や config file detection の欠落として切り分ける。 |
| `ghqr scan --repository ORG/REPO` | `Administration: Read-only` がない | FGPAT permission不足 | https://docs.github.com/en/rest/authentication/permissions-required-for-fine-grained-personal-access-tokens | ruleset / branch protection enrichment の失敗として切り分ける。 |
| `ghqr scan --organization ORG` | Resource owner / Repository access / org approval が対象 org と一致しない | FGPAT設定不足 | https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens | 初期検証では禁止。repository 明示指定で先に切り分ける。 |
| organization auto-discovery | `GET /user/orgs` または GraphQL organization visibility が空になる | GitHub visibility / org policy | https://docs.github.com/en/rest/authentication/endpoints-available-for-fine-grained-personal-access-tokens | `--repository` または `--organization` 明示指定で切り分ける。 |
| organization repositories GraphQL | repository 数が多い | GitHub API制限 | https://docs.github.com/en/graphql/overview/rate-limits-and-query-limits-for-the-graphql-api | API volume / GraphQL cost の観点で段階的検証にする。 |
| `ghqr scan --enterprise ENT` | Enterprise role / scope / endpoint token type が不足する | Enterprise制限 | https://docs.github.com/en/enterprise-cloud@latest/rest/enterprise-admin/audit-log | 初期検証では禁止。Enterprise owner と合意してから限定 probe にする。 |
| Enterprise audit log | FGPAT PAT が endpoint の Fine-grained access token types に含まれない | GitHub API制限 | https://docs.github.com/en/enterprise-cloud@latest/rest/enterprise-admin/audit-log | Classic PAT または GitHub App token が必要な領域として文書化する。 |
| Enterprise code security settings | ghqr 実装 path の FGPAT 対応が公式 Docs で未確認 | 未確認 | https://docs.github.com/en/rest/code-security/configurations | Enterprise 契約環境で限定 probe が必要。 |
| Copilot billing | Copilot 契約または organization owner 権限がない | 契約 / permission不足 | https://docs.github.com/en/rest/copilot/copilot-user-management | `403` / `404` / empty result を contract / role と分けて記録する。 |
| Dependabot alerts | Dependabot / GHAS / alerts permission がない | 契約 / permission不足 | https://docs.github.com/en/rest/authentication/permissions-required-for-fine-grained-personal-access-tokens | Security endpoint は product availability と permission を分けて記録する。 |
| Code scanning alerts | GHAS / code scanning / alerts permission がない | 契約 / permission不足 | https://docs.github.com/en/rest/authentication/permissions-required-for-fine-grained-personal-access-tokens | `403` / `404` / empty result を区別する。 |
| Secret scanning alerts | GHAS / secret scanning / alerts permission がない | 契約 / permission不足 | https://docs.github.com/en/rest/authentication/permissions-required-for-fine-grained-personal-access-tokens | `403` / `404` / empty result を区別する。 |
| Security managers | Organization admin / security manager visibility がない | permission不足 | https://docs.github.com/en/rest/authentication/endpoints-available-for-fine-grained-personal-access-tokens | ghqr は inaccessible endpoint を empty result として扱う箇所がある。 |
| Archived repository | archived 特有の read-only 状態で scan 結果が private test repository と異なる | 未確認 | 未確認 | Runbook の別 repository 検証対象にする。 |
| Disabled repository | GitHub 管理側状態により API visibility が変わる | 未確認 | 未確認 | Enterprise 管理者が確認できる場合だけ検証する。 |
| Transferred repository | Resource owner / repository full name が移管後 owner と一致しない | FGPAT設定不足 | https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens | local config の target repository を移管後 owner に更新する。 |

## 6. GraphQL API 差分

GitHub GraphQL docs は、personal access token で GraphQL API に認証でき、要求する data によって必要 scope / permission が決まると説明している。GraphQL field ごとの FGPAT 対応差分は、この資料で確認した公式 Docs には REST endpoint のような一覧がないため、field 単位の FGPAT 対応は `未確認` とする。

| ghqr機能 | GraphQL query/field | Classic PAT | FGPAT | 必要権限 | 根拠URL | 実測状況 | 備考 |
|---|---|---|---|---|---|---|---|
| authentication smoke check | `viewer` | 可能 | 可能 | 要求 data に応じる | https://docs.github.com/en/graphql/guides/forming-calls-with-graphql | 未確認 | ghqr は initialization で REST `GET /user` を使う。 |
| enterprise discovery | `viewer.enterprises` | 未確認 | 未確認 | Enterprise visibility / permission。field 単位 Docs 未確認 | https://docs.github.com/en/graphql/guides/forming-calls-with-graphql | 既存実測で両 token とも enterprise scope / permission 不足 warning | ghqr auto-discovery に影響。 |
| organization repository enumeration | `organization(login:)` | 可能 | 可能 | Organization visibility / approval / repository access | https://docs.github.com/en/graphql/guides/forming-calls-with-graphql | Classic PAT / FGPAT とも成功済み | ghqr organization scan で使用。 |
| organization repository enumeration | `organization.repositories` | 可能 | 可能 | Repository access / organization access | https://docs.github.com/en/graphql/guides/forming-calls-with-graphql | Classic PAT / FGPAT とも成功済み | Organization 全体 scan は初期検証禁止。 |
| explicit repository scan | `repository(owner:, name:)` | 可能 | 可能 | Repository visibility / selected repository access | https://docs.github.com/en/graphql/guides/forming-calls-with-graphql | Classic PAT / FGPAT とも成功済み | FGPAT Resource owner 誤りでは `404` / resolve failure になる。 |
| ruleset | `repository.rulesets(first:, includeParents:)` | 可能 | 可能 | Repository Administration read が必要な範囲。GraphQL field 単位 Docs 未確認 | https://docs.github.com/en/graphql/guides/forming-calls-with-graphql | FGPAT で成功済み | ghqr は repository rulesets を GraphQL raw POST で取得。 |
| branch protection | `defaultBranchRef.branchProtectionRule` | 可能 | 可能 | Repository visibility / branch protection visibility。field 単位 Docs 未確認 | https://docs.github.com/en/graphql/guides/forming-calls-with-graphql | FGPAT で成功済み | Batch scan でも使用。 |
| Dependabot alerts | `vulnerabilityAlerts` | 可能 | 可能 | Repository vulnerability alerts visibility。field 単位 Docs 未確認 | https://docs.github.com/en/graphql/guides/forming-calls-with-graphql | Explicit repository scan で成功済み | Org batch scan では complexity 回避のため一部 connection を省略。 |
| collaborators | `collaborators` | 可能 | 可能 | Repository collaborator visibility。field 単位 Docs 未確認 | https://docs.github.com/en/graphql/guides/forming-calls-with-graphql | Explicit repository scan で成功済み | Org batch scan では complexity 回避のため省略。 |
| deploy keys | `deployKeys` | 可能 | 可能 | Repository deploy key visibility。field 単位 Docs 未確認 | https://docs.github.com/en/graphql/guides/forming-calls-with-graphql | Explicit repository scan で成功済み | Org batch scan では complexity 回避のため省略。 |
| enterprise scan | `enterprise(slug:)` | 未確認 | 未確認 | Enterprise visibility / permission。field 単位 Docs 未確認 | https://docs.github.com/en/graphql/guides/forming-calls-with-graphql | 既存実測で両 token とも権限不足 warning | Enterprise 全体 scan は初期検証禁止。 |
| enterprise organization enumeration | `enterprise.organizations` | 未確認 | 未確認 | Enterprise visibility / permission。field 単位 Docs 未確認 | https://docs.github.com/en/graphql/guides/forming-calls-with-graphql | 既存実測で両 token とも権限不足 warning | API volume が増えるため段階的検証対象。 |
| EMU status | `enterprise.ownerInfo.samlIdentityProvider` | 未確認 | 未確認 | Enterprise owner info visibility。field 単位 Docs 未確認 | https://docs.github.com/en/graphql/guides/forming-calls-with-graphql | 既存実測で権限不足時は debug / nil 扱い | ghqr は org scan でも EMU 判定に使う。 |

## 7. organization policy / approval 差分

| 操作/制御 | Classic PAT | FGPAT | 根拠URL | 原文要約 | 実務上の注意 |
|---|---|---|---|---|---|
| Organization resource access policy | Organization owner が personal access tokens (classic) または FGPAT の access policy を設定可能 | Organization owner が personal access tokens (classic) または FGPAT の access policy を設定可能 | https://docs.github.com/en/organizations/managing-programmatic-access-to-your-organization/setting-a-personal-access-token-policy-for-your-organization | Restrict access / Allow access を token type ごとに設定できる。 | Org policy が restrict の場合、token 設定が正しくても API が拒否される。 |
| Public resource access | Public resources within organization には policy に関わらず access できると Docs が説明 | Public resources within organization には approval 前でも read できると Docs が説明 | https://docs.github.com/en/organizations/managing-programmatic-access-to-your-organization/setting-a-personal-access-token-policy-for-your-organization | Personal access tokens will have access to public resources within the organization. FGPAT can read public resources without approval. | Private repo 検証とは分けて記録する。 |
| FGPAT approval policy | 対象外。Classic PAT は approval policy の対象ではない | 対象。Require administrator approval の場合、organization owner approval が必要 | https://docs.github.com/en/organizations/managing-programmatic-access-to-your-organization/setting-a-personal-access-token-policy-for-your-organization | Only fine-grained personal access tokens, not personal access tokens (classic), are subject to approval. | FGPAT の private resource access 失敗時は approval status を確認する。 |
| Pending request review | 未確認 | Organization owner が approve / deny できる | https://docs.github.com/en/organizations/managing-programmatic-access-to-your-organization/managing-requests-for-personal-access-tokens-in-your-organization | Organization owners can approve or deny FGPAT requests. | Runbook に human-only step として残す。 |
| Maximum lifetime policy | Organization owner が policy を設定可能。Classic PAT には expiration requirement がないと Docs が説明 | Organization owner が policy を設定可能。FGPAT default maximum lifetime policy は 366 days と Docs が説明 | https://docs.github.com/en/organizations/managing-programmatic-access-to-your-organization/setting-a-personal-access-token-policy-for-your-organization | Tokens with non-compliant lifetimes are blocked from organization resources. | 長期検証では token expiration を記録する。ただし token 値は記録しない。 |

## 8. ghqr への影響

| ghqr機能 | 使用API | Classic PAT | FGPAT | 実測結果 | 必要permission | リスク | README記載要否 |
|---|---|---|---|---|---|---|---|
| token initialization | `GH_TOKEN`、`GITHUB_TOKEN`、REST `GET /user` | 使用可能 | 使用可能 | 両方成功済み | User authentication | token 種別判定なし。権限不足は後続 API で発生する。 | 必要 |
| explicit repository scan | GraphQL `repository(owner:, name:)`、repository fields、rulesets | 使用可能 | 使用可能 | 両方 success、reports generated | Resource owner / selected repository / Metadata / Contents / Administration read | Resource owner 誤りや repository access 不足で `404` / GraphQL resolve failure | 必要 |
| organization scan | REST org endpoints、GraphQL `organization.repositories` | 使用可能 | 使用可能 | 既存限定 org で両方 success | Organization access / repository access / org approval | 初期検証で全 repo scan を実施すると API volume と audit visibility が増える | 必要 |
| organization auto-discovery | REST `GET /user/orgs`、GraphQL org query | 使用可能 | 使用可能 | 既存限定環境で両方 success | User/org visibility | FGPAT の Resource owner / approval / repository access を確認する。 | 必要 |
| enterprise scan | GraphQL `enterprise(slug:)`、REST enterprise endpoints | Enterprise admin / scope が必要 | FGPAT PAT では一部 endpoint 非対応または未確認 | 既存実測では両 token とも権限不足 warning | Enterprise role / `read:enterprise` / endpoint-specific permission | Enterprise 全体 scan は初期検証禁止。Audit / security logs で目立つ | 必要 |
| enterprise auto-discovery | GraphQL `viewer.enterprises` | 未確認 | 未確認 | 既存実測では両 token とも権限不足 warning | Enterprise visibility / scope | 既存実測では enterprise discovery 失敗後も repository 明示指定 scan は成功した。 | 必要 |
| ruleset | GraphQL `repository.rulesets` | 使用可能 | 使用可能 | FGPAT で success | Repository Administration read が必要な範囲 | Permission 不足では ruleset enrichment failure になる | 必要 |
| branch protection | GraphQL `defaultBranchRef.branchProtectionRule` | 使用可能 | 使用可能 | FGPAT で success | Repository visibility | Ruleset と legacy branch protection の差を README で補足 | 任意 |
| collaborators | GraphQL `collaborators` | 使用可能 | 使用可能 | Explicit repository scan で success | Repository collaborator visibility | Org batch scan では complexity 回避で省略 | 必要 |
| deploy keys | GraphQL `deployKeys` | 使用可能 | 使用可能 | Explicit repository scan で success | Repository deploy key visibility | Org batch scan では complexity 回避で省略 | 必要 |
| Dependabot alerts | GraphQL `vulnerabilityAlerts`、REST `dependabot/alerts` | 使用可能 | 使用可能 | REST 単体は両方 success | Dependabot alerts read / product availability | GHAS / Dependabot availability と permission 不足の切り分けが必要 | 必要 |
| code scanning alerts | REST `code-scanning/alerts` | 使用可能 | 使用可能 | REST 単体は両方 success | Code scanning alerts read / product availability | GHAS / code scanning availability と permission 不足の切り分けが必要 | 必要 |
| secret scanning alerts | REST `secret-scanning/alerts` | 使用可能 | 使用可能 | REST 単体は両方 success | Secret scanning alerts read / product availability | GHAS / secret scanning availability と permission 不足の切り分けが必要 | 必要 |
| Copilot | REST `GET /orgs/{org}/copilot/billing` | `manage_billing:copilot` または `read:org` | Copilot Business org read または Administration org read | REST 単体は両方 success | Organization owner / Copilot Business permission | Docs は organization owners only と説明している。 | 必要 |
| audit log | REST `GET /enterprises/{enterprise}/audit-log` | Enterprise admin + `read:audit_log` | FGPAT PAT は非対応として扱う | 既存実測では両 token とも `403` | Enterprise admin / endpoint token type | FGPAT では Classic PAT と同等にできない項目として文書化 | 必要 |
| report rendering | local JSON / Markdown / Excel rendering | token 種別に依存しない | token 種別に依存しない | 両方 success | API scan results が前提 | Raw reports は commit 禁止 | 必要 |

## 8.1 既存実測結果の統合サマリ

既存検証は、公開 fork に載せるため実 organization / repository / username を placeholder 化している。新規の organization-wide scan / enterprise-wide scan はこの再設計では実施していない。

| Feature | Classic PAT result | FGPAT result | Same behavior? | Classification |
|---|---|---|---|---|
| explicit repository scan | success、exit code 0、reports generated | success、exit code 0、reports generated | Yes | 両方可能 |
| organization scan | success、exit code 0、限定 org / repo で完了 | success、exit code 0、限定 org / repo で完了 | Yes | 両方可能 |
| organization auto-discovery | 限定 org を検出 | 限定 org を検出 | Yes in this validation | 両方可能 |
| enterprise discovery | `read:enterprise` 不足 warning | `read:enterprise` 不足 warning | Yes | Enterprise permission limitation |
| enterprise scan | enterprise settings は失敗、後続 org scan は成功 | enterprise settings は失敗、後続 org scan は成功 | Yes | Enterprise permission limitation |
| audit log | REST 単体 `403` | REST 単体 `403` | Yes | Enterprise permission / token type limitation |
| Copilot | REST 単体 exit code 0 | REST 単体 exit code 0 | Yes | 両方可能 |
| security endpoints | REST 単体 exit code 0 | REST 単体 exit code 0 | Yes | 両方可能 |
| org actions permissions | REST 単体 `403` | REST 単体 `403` | Yes | Permission limitation |
| ruleset / branch protection | scan 内で success | scan 内で success | Yes | 両方可能 |
| REST / GraphQL 単体成功だが ghqr だけ失敗 | 未検出 | 未検出 | N/A | ghqr-only failure なし |

重要な因果関係:

- 最初の FGPAT 失敗は、token 自体ではなく Resource owner / Repository access が対象 organization repository と一致していなかったことによる。
- Resource owner を対象 organization にし、対象 repository を Repository access に含め、Metadata / Contents / Administration read を付与した FGPAT では repository visibility と `ghqr scan --repository ORG/REPO` が成功した。
- Enterprise discovery / audit log / enterprise settings の失敗は、REST / GraphQL 単体でも同じ制限が出ており、現時点では ghqr-only failure ではない。

## 8.2 PR / Issue draft

### PR draft

`ghqr` の Fine-grained PAT 対応について、現時点の調査では explicit repository scan と限定 organization scan は適切な token 設定で正常動作することを確認した。

Fine-grained PAT は Classic PAT の完全上位互換ではない。GitHub API は endpoint ごとの permission、Resource owner、Repository access、organization approval policy、enterprise role、product availability の影響を受ける。

今回の調査では、最初の失敗は `ghqr` の不具合ではなく、Fine-grained PAT の Resource owner と Repository access 設定不整合により説明できた。その後、対象 organization を Resource owner にし、対象 repository を明示した Fine-grained PAT では、repository visibility の REST API と `ghqr scan --repository` の両方が成功した。

現時点では必須の機能修正ではなく、README、warning、troubleshooting を改善し、Fine-grained PAT の前提条件と制約を明確にする方針が妥当である。

### Issue bullets

- FGPAT でも explicit repository scan は成功済み。
- Classic PAT と完全同一挙動は保証しない。
- Resource owner、Repository access、permissions、organization approval policy が重要。
- Enterprise / audit / Copilot / security endpoint は追加 permission、GitHub API 制限、GitHub product availability に依存する。
- 現時点で REST / GraphQL 単体は成功するが ghqr だけ失敗する箇所は確認されていない。
- README / warning / troubleshooting を先に改善するのが妥当。

## 9. 実測計画

安全条件:

- organization 全体 scan は禁止
- enterprise 全体 scan は禁止
- repository 明示指定のみ
- 目立たない検証 repository だけを使う
- token 値は記録しない
- `403` / `404` / warning / empty result を記録する
- audit log で見られても説明できる範囲に限定する

実測対象:

| 検証ID | token種別 | 設定 | コマンド | 期待結果 | 実際の結果 | exit code | HTTP status | 分類 | 根拠 |
|---|---|---|---|---|---|---|---|---|---|
| T01 | Classic PAT | repository read 可能 | `gh api repos/ORG/REPO --jq .full_name` | `ORG/REPO` が返る | 既存実測で成功 | 0 | 200 | 両方可能 | この資料の既存実測サマリ |
| T02 | FGPAT | Resource owner が ORG、Repository access が REPO | `GH_CONFIG_DIR=/tmp/gh-fgpat-validation gh api repos/ORG/REPO --jq .full_name` | `ORG/REPO` が返る | 既存実測で成功 | 0 | 200 | 両方可能 | この資料の既存実測サマリ |
| T03 | Classic PAT | repository read 可能 | `GITHUB_TOKEN="$(GH_CONFIG_DIR=/tmp/gh-classic-validation gh auth token)" ghqr scan --repository ORG/REPO --output-name docs/research/fgpat/results/classic-repo` | scan completed、reports generated | 既存実測で成功 | 0 | N/A | 両方可能 | この資料の既存実測サマリ |
| T04 | FGPAT | Resource owner が ORG、Repository access が REPO、必要 permission 付与済み | `GITHUB_TOKEN="$(GH_CONFIG_DIR=/tmp/gh-fgpat-validation gh auth token)" ghqr scan --repository ORG/REPO --output-name docs/research/fgpat/results/fgpat-repo` | scan completed、reports generated | 既存実測で成功 | 0 | N/A | 両方可能 | この資料の既存実測サマリ |
| T05 | FGPAT | permission を 1 つ外す | `GITHUB_TOKEN="$(GH_CONFIG_DIR=/tmp/gh-fgpat-missing-permission gh auth token)" ghqr scan --repository ORG/REPO --output-name docs/research/fgpat/results/fgpat-missing-permission` | permission 不足の `403` / warning | 未実施 | 未確認 | 未確認 | 未確認 | Runbook に従い human approval 後に実施 |
| T06 | FGPAT | Resource owner が対象 ORG ではない | `GH_CONFIG_DIR=/tmp/gh-fgpat-wrong-owner gh api repos/ORG/REPO --jq .full_name` | `404` | 既存実測で `404` | 1 | 404 | FGPAT設定不足 | この資料の既存実測サマリ |
| T07 | FGPAT | Organization approval 未承認 | `GH_CONFIG_DIR=/tmp/gh-fgpat-pending gh api repos/ORG/REPO --jq .full_name` | private resource access 不可 | 未実施 | 未確認 | 未確認 | 未確認 | https://docs.github.com/en/organizations/managing-programmatic-access-to-your-organization/managing-requests-for-personal-access-tokens-in-your-organization |

## 10. 最終判定

### Classic PAT のみと確認できたもの

- Outside collaborator が collaborator である organization repository にアクセスする場合は Classic PAT のみ。
  - 出典: https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens
- 自分または member organization が owner ではない public repository への write access は Classic PAT のみ。
  - 出典: https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens
- `GET /enterprises/{enterprise}/audit-log/stream-key` は FGPAT 非対応。
  - 出典: https://docs.github.com/en/enterprise-cloud@latest/rest/enterprise-admin/audit-log
- `GET /enterprises/{enterprise}/audit-log/streams` は FGPAT 非対応。
  - 出典: https://docs.github.com/en/enterprise-cloud@latest/rest/enterprise-admin/audit-log
- `GET /enterprises/{enterprise}/code-security/configurations` は FGPAT 非対応。
  - 出典: https://docs.github.com/en/rest/code-security/configurations
- `POST /enterprises/{enterprise}/code-security/configurations` は FGPAT 非対応。
  - 出典: https://docs.github.com/en/rest/code-security/configurations

### FGPAT のみ、または FGPAT の固有制御と確認できたもの

- Resource owner を指定できる。
  - 出典: https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens
- Repository access を All repositories または Only select repositories として限定できる。
  - 出典: https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens
- Repository / Organization / Account permission を endpoint に応じて細かく設定できる。
  - 出典: https://docs.github.com/en/rest/authentication/permissions-required-for-fine-grained-personal-access-tokens
- Organization owner が FGPAT request を approve / deny できる。
  - 出典: https://docs.github.com/en/organizations/managing-programmatic-access-to-your-organization/managing-requests-for-personal-access-tokens-in-your-organization
- FGPAT approval policy は FGPAT のみ対象であり、Classic PAT は対象外。
  - 出典: https://docs.github.com/en/organizations/managing-programmatic-access-to-your-organization/setting-a-personal-access-token-policy-for-your-organization

### 両方可能だが条件が違うもの

- REST API の多くの endpoint は両方で利用できるが、Classic PAT は scope、FGPAT は endpoint ごとの permission / Resource owner / Repository access / org approval が条件になる。
  - 出典: https://docs.github.com/en/rest/authentication/endpoints-available-for-fine-grained-personal-access-tokens
  - 出典: https://docs.github.com/en/rest/authentication/permissions-required-for-fine-grained-personal-access-tokens
- GraphQL API は personal access token で認証できるが、要求する data によって必要 scope / permission が決まる。
  - 出典: https://docs.github.com/en/graphql/guides/forming-calls-with-graphql
- `GET /orgs/{org}/copilot/billing` は両方で可能だが、Classic PAT は `manage_billing:copilot` または `read:org`、FGPAT は `GitHub Copilot Business` organization read または `Administration` organization read が必要。
  - 出典: https://docs.github.com/en/rest/copilot/copilot-user-management
- ghqr の explicit repository scan は既存実測で両方成功したが、FGPAT は Resource owner / Repository access / permission / approval が正しい場合に限る。
  - 出典: この資料の既存実測サマリ

### 未確認のもの

- GraphQL field 単位で、Classic PAT と FGPAT の permission 差を公式 Docs から網羅確認すること。
  - 未確認理由: REST endpoint のような FGPAT endpoint / permission matrix を、この資料で参照した GraphQL Docs では確認できなかった。
  - 次に見る Docs / 方法: GitHub GraphQL schema docs の各 object / field、GraphQL 実測 probe。
- `GET /enterprises/{enterprise}/dependabot/alerts`、`GET /enterprises/{enterprise}/code-scanning/alerts`、`GET /enterprises/{enterprise}/secret-scanning/alerts` の FGPAT PAT 対応。
  - 未確認理由: この資料の公式 Docs 確認では endpoint 個別 permission を確定できていない。
  - 次に見る Docs / 方法: endpoint 個別 Docs と Enterprise 契約環境での限定 probe。
- `GET /enterprises/{enterprise}/code_security/settings` の FGPAT PAT 対応。
  - 未確認理由: GitHub Docs で確認できた code security configurations endpoint と ghqr 実装 path が異なるため、転用しない。
  - 次に見る Docs / 方法: endpoint 個別 Docs または Enterprise 契約環境での限定 probe。
- Organization approval 未承認状態の実測。
  - 未確認理由: approval 未承認 token を安全に作成する human-only step が必要。
  - 次に見る Docs / 方法: `enterprise-validation-runbook.md` の safe staged validation policy に従って実施。

### ghqr README に反映すべきこと

- FGPAT は利用可能。
- ただし Classic PAT の完全上位互換ではない。
- Resource owner / Repository access / permissions / organization approval が重要。
- Organization / Enterprise / audit / Copilot / security endpoint は追加制約がある。
- 初期検証は `ghqr scan --repository ORG/REPO` から始める。
- 初期検証で `ghqr scan --organization ORG_NAME` は実施しない。
- FGPAT の `404` は repository 不在だけでなく Resource owner / Repository access / approval の問題でも起きる。
- FGPAT の `403` は endpoint permission 不足、organization policy、enterprise role、product availability の切り分けが必要。

### ghqr 実装に反映すべきこと

- 必須修正:
  - 現時点では確認なし。既存実測では REST / GraphQL 単体では成功するが ghqr だけ失敗する箇所は確認されていない。
- Warning 改善:
  - FGPAT 利用時の `403` / `404` に対して、Resource owner、Repository access、permission、organization approval、enterprise role を確認する案内を追加する余地がある。
  - Enterprise audit log / enterprise code security configuration のように FGPAT PAT 非対応 endpoint が公式 Docs で確認できる範囲は、Classic PAT または GitHub App token が必要である旨を warning / troubleshooting に書く余地がある。
- README 修正:
  - 必要。README / troubleshooting で FGPAT の使い方、制約、安全な段階的検証順序を説明する。
