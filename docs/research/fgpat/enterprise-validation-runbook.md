# Enterprise FGPAT validation runbook

この runbook は、GitHub Enterprise 契約ユーザーが `ghqr` の Classic PAT と Fine-grained PAT (FGPAT) の挙動差を安全に検証するための手順です。

この資料では実 organization、enterprise、username、repository 名を記録しません。記録時は `example-enterprise`、`example-org`、`example-repo` のような placeholder に置換してください。

## 1. GitHub Enterprise prerequisites

検証前に、Enterprise owner または Organization owner が次を確認します。

### Enterprise environment

- GitHub Enterprise Cloud か GitHub Enterprise Server か
- Enterprise Managed Users (EMU) を利用しているか
- Enterprise slug を検証資料に記録してよいか
- Enterprise audit log へのアクセス権限を持つ管理者がいるか
- Enterprise-level API access に必要な role を持つ検証ユーザーがいるか

### Authentication and access policy

- SAML enforcement が有効か
- SSO authorization が Classic PAT と FGPAT の両方で必要か
- Enterprise または Organization で PAT policy が有効か
- Classic PAT が許可されているか
- Fine-grained PAT が許可されているか
- Fine-grained PAT approval policy が有効か
- IP allow list が API access に影響するか
- 検証環境の実行元 IP が許可済みか

### Product features

- GitHub Advanced Security (GHAS) 契約があるか
- Code scanning が利用可能か
- Secret scanning が利用可能か
- Dependabot alerts が利用可能か
- GitHub Copilot Business / Enterprise 契約があるか
- Organization-level Actions policies を管理できるか

## 2. Fine-grained PAT creation steps

GitHub Web UI で検証用 FGPAT を作成します。token 値は記録しません。

1. GitHub に検証ユーザーでサインインする
2. 右上の avatar menu を開く
3. `Settings` を開く
4. 左メニュー下部の `Developer settings` を開く
5. `Personal access tokens` を開く
6. `Fine-grained tokens` を開く
7. `Generate new token` を選択する
8. `Token name` に検証用であることが分かる名前を入れる
9. `Expiration` を検証期間に合わせて短めに設定する
10. `Resource owner` で検証対象 organization を選ぶ
11. `Repository access` を選ぶ
12. 最初は `Only select repositories` を推奨し、検証用 repository だけを選ぶ
13. 必要な `Repository permissions` を設定する
14. 必要な `Organization permissions` を設定する
15. 内容を確認して token を生成する
16. token 値は画面から安全な secret manager または検証 shell のみへ渡す
17. token 値をチャット、issue、PR、スクリーンショット、ログに残さない

## 3. Recommended permissions

最小権限から始め、失敗した endpoint ごとに permission を追加します。

### Repository scan baseline

- Repository access:
  - `Only select repositories`
  - 検証用 private repository を 1 つ選択
- Repository permissions:
  - `Metadata: Read-only`
  - `Contents: Read-only`
  - `Administration: Read-only`

### Organization scan

- Organization permissions:
  - `Members: Read-only` が必要になる可能性
  - `Administration: Read-only` が必要になる可能性
  - `Actions policies: Read-only` が org Actions permissions endpoint で必要になる可能性
- Repository permissions:
  - Organization 配下の検証対象 repository を含める
  - Security endpoint 検証時は各 security permission を追加する

### Security endpoints

- Dependabot alerts:
  - Dependabot alerts related read permission
- Code scanning:
  - Code scanning alerts related read permission
- Secret scanning:
  - Secret scanning alerts related read permission
- GHAS:
  - GHAS 契約と repository / org policy が必要

### Copilot

- Copilot Business / Enterprise 契約を確認する
- Organization Copilot billing / seat information を読める role を確認する
- Copilot endpoint が plan や role に依存することを記録する

### Enterprise endpoints

- Enterprise discovery:
  - `read:enterprise` 相当の権限が必要
- Enterprise scan:
  - Enterprise settings / organizations を読める Enterprise role が必要
- Audit log:
  - `admin:enterprise` 相当の権限が必要
- Enterprise GHAS settings:
  - Enterprise-level code security settings を読める権限が必要

## 4. Organization owner approval steps

Organization owner は、FGPAT が organization resources にアクセスできる状態か確認します。

### Request approval

1. Organization の `Settings` を開く
2. `Personal access tokens` または `Third-party access` related section を開く
3. `Fine-grained personal access tokens` の request 一覧を確認する
4. 検証用 token request を開く
5. Resource owner、repository access、requested permissions を確認する
6. 検証目的と期間が妥当なら approve する
7. 承認日時と承認者 role を記録する

### Token review

- token が検証対象 repository を含んでいるか確認する
- token が必要な organization permissions を含んでいるか確認する
- token が期限切れでないか確認する
- token が SSO authorization を必要とする場合は authorize 済みか確認する

### Policy settings

- Classic PAT が許可されているか
- Fine-grained PAT が許可されているか
- approval required の有無
- IP allow list enforcement の有無
- SAML / SSO enforcement の有無

## 5. Validation repository preparation

検証用 repository は公開情報を含まない private repository とします。

### Repository baseline

- private repository を作成する
- README だけの最小 repository から開始する
- default branch を確認する
- archived repository の検証が必要な場合は別 repository を作る
- public repository 差分が必要な場合は別 repository を作る

### Branch protection and rulesets

- legacy branch protection を 1 つ設定する
- repository ruleset を 1 つ設定する
- required pull request review を有効にする
- required status checks を 1 つ設定する
- force push / deletion policy を確認する

### Security features

- Dependabot alerts を有効にする
- Dependabot security updates を有効にする
- Code scanning を有効にする
- Secret scanning を有効にする
- Secret scanning push protection を有効にする
- GHAS 契約が必要な設定は契約状態を記録する

### Organization features

- Copilot billing / seat assignment を確認する
- Actions permissions を organization level で確認する
- Security manager team を確認する
- Organization default repository permission を確認する

## 6. Enterprise validation matrix

Classic PAT と FGPAT で同じ対象、同じ `ghqr` command、同じ output name prefix を使って比較します。

| Validation item | Classic PAT command | FGPAT command | Expected classification |
| --- | --- | --- | --- |
| repository scan | `ghqr scan --repository example-org/example-repo` | same command with FGPAT token | same behavior or permission limitation |
| organization scan | `ghqr scan --organization example-org` | same command with FGPAT token | same behavior or org policy limitation |
| auto-discovery | `ghqr scan` | same command with FGPAT token | same behavior or discovery limitation |
| enterprise discovery | `ghqr scan` | same command with FGPAT token | enterprise limitation |
| enterprise scan | `ghqr scan --enterprise example-enterprise` | same command with FGPAT token | enterprise limitation |
| audit log | `gh api enterprises/example-enterprise/audit-log` | same endpoint with FGPAT config | enterprise limitation |
| Copilot | `gh api orgs/example-org/copilot/billing` | same endpoint with FGPAT config | same behavior or permission limitation |
| security endpoints | Dependabot / code scanning / secret scanning endpoints | same endpoints with FGPAT config | same behavior or permission limitation |
| rulesets | repository scan output | repository scan output | same behavior or permission limitation |
| branch protection | repository scan output | repository scan output | same behavior or permission limitation |
| actions permissions | `gh api orgs/example-org/actions/permissions` | same endpoint with FGPAT config | permission limitation |

## 7. Safe command examples

Do not paste token values into docs, issues, PRs, chat, or shell history.

### Isolated gh auth stores

```bash
mkdir -p /tmp/gh-classic-validation
mkdir -p /tmp/gh-fgpat-validation
```

### Login with token from standard input

Run these only in a shell where the relevant variable already exists.

```bash
printf '%s\n' "$CLASSIC_GH_TOKEN" | GH_CONFIG_DIR=/tmp/gh-classic-validation gh auth login --hostname github.com --with-token
printf '%s\n' "$FGPAT_GH_TOKEN" | GH_CONFIG_DIR=/tmp/gh-fgpat-validation gh auth login --hostname github.com --with-token
```

### Repository visibility checks

```bash
GH_CONFIG_DIR=/tmp/gh-classic-validation gh api repos/example-org/example-repo --jq .full_name
GH_CONFIG_DIR=/tmp/gh-fgpat-validation gh api repos/example-org/example-repo --jq .full_name
```

### ghqr scan commands

```bash
GITHUB_TOKEN="$(GH_CONFIG_DIR=/tmp/gh-classic-validation gh auth token)" ghqr scan --repository example-org/example-repo --output-name audit_data/classic-repo
GITHUB_TOKEN="$(GH_CONFIG_DIR=/tmp/gh-fgpat-validation gh auth token)" ghqr scan --repository example-org/example-repo --output-name audit_data/fgpat-repo
```

```bash
GITHUB_TOKEN="$(GH_CONFIG_DIR=/tmp/gh-classic-validation gh auth token)" ghqr scan --organization example-org --output-name audit_data/classic-org
GITHUB_TOKEN="$(GH_CONFIG_DIR=/tmp/gh-fgpat-validation gh auth token)" ghqr scan --organization example-org --output-name audit_data/fgpat-org
```

```bash
GITHUB_TOKEN="$(GH_CONFIG_DIR=/tmp/gh-classic-validation gh auth token)" ghqr scan --enterprise example-enterprise --output-name audit_data/classic-enterprise
GITHUB_TOKEN="$(GH_CONFIG_DIR=/tmp/gh-fgpat-validation gh auth token)" ghqr scan --enterprise example-enterprise --output-name audit_data/fgpat-enterprise
```

### REST endpoint probes

```bash
GH_CONFIG_DIR=/tmp/gh-classic-validation gh api --silent orgs/example-org/actions/permissions
GH_CONFIG_DIR=/tmp/gh-fgpat-validation gh api --silent orgs/example-org/actions/permissions
```

```bash
GH_CONFIG_DIR=/tmp/gh-classic-validation gh api --silent orgs/example-org/copilot/billing
GH_CONFIG_DIR=/tmp/gh-fgpat-validation gh api --silent orgs/example-org/copilot/billing
```

```bash
GH_CONFIG_DIR=/tmp/gh-classic-validation gh api --silent enterprises/example-enterprise/audit-log?include=all\&per_page=1
GH_CONFIG_DIR=/tmp/gh-fgpat-validation gh api --silent enterprises/example-enterprise/audit-log?include=all\&per_page=1
```

### GraphQL probes

```bash
GH_CONFIG_DIR=/tmp/gh-classic-validation gh api graphql \
  -f query='query($owner:String!,$name:String!){ repository(owner:$owner,name:$name){ name isPrivate defaultBranchRef { name } } }' \
  -f owner=example-org \
  -f name=example-repo \
  --jq .data.repository.name
```

```bash
GH_CONFIG_DIR=/tmp/gh-fgpat-validation gh api graphql \
  -f query='query($org:String!){ organization(login:$org){ login repositories(first:10){ totalCount nodes { name } } } }' \
  -f org=example-org \
  --jq .data.organization.repositories.totalCount
```

## 8. Recording rules

Record every validation step with:

- command
- execution session
- token source label only
- exit code
- stdout summary
- stderr summary
- generated files
- classification
- next action if failed

Do not record:

- token values
- token fragments
- token prefixes or suffixes
- raw `gh auth status` token line
- screenshots containing personal data
- real enterprise slug unless approved for public disclosure
- internal URLs
- IP addresses
- shell history
- `.env`
- `.codex`
- `GH_CONFIG_DIR` contents
- generated reports that may include private organization data

## 9. Anonymization rules

- Replace real organization names with `example-org`
- Replace real enterprise slug with `example-enterprise`
- Replace real repository names with `example-repo`
- Replace real usernames with `example-user`
- Replace internal hostnames with `example.internal`
- Replace IP addresses with `example-ip` or omit them
- Keep GitHub official documentation URLs as-is

## 10. Report storage

- Store raw generated reports outside public docs unless they are sanitized
- Do not commit JSON / Markdown / Excel reports from real organizations
- If a report must be shared, remove organization names, usernames, repository names, URLs, IDs, and any security findings that identify internal systems
- Prefer summarized tables over raw report files

## 11. Expected result classification

Use exactly one primary classification for each result.

- `same behavior`
  - Classic PAT and FGPAT both succeed or fail in the same way
- `GitHub permission limitation`
  - Endpoint requires a permission not granted to the token
- `org policy limitation`
  - Organization approval, SSO, SAML, IP allow list, or PAT policy blocks access
- `enterprise limitation`
  - Enterprise role, enterprise permission, contract, or plan blocks access
- `ghqr-only failure`
  - REST / GraphQL standalone probe succeeds, but `ghqr` fails for the same resource

## 12. Follow-up for upstream PR

After Enterprise validation is complete, split upstream-facing changes into:

- README guidance for FGPAT setup
- troubleshooting for 403 / 404 / empty discovery results
- warning improvements for discovery and endpoint-level permission failures
- no raw research logs
- no generated reports
- no organization-specific command output
