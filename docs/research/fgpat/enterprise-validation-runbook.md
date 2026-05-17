# Enterprise FGPAT validation runbook

この runbook は、GitHub Enterprise 契約ユーザーが `ghqr` の Classic PAT と Fine-grained PAT (FGPAT) の挙動差を安全に検証するための手順です。

この資料では実 organization、enterprise、username、repository 名を記録しません。記録時は `example-enterprise`、`example-org`、`example-repo` のような placeholder に置換してください。

## 0. Operating model

この検証は、最初に人間が GitHub Web UI で準備し、その後にコーディングエージェントへローカル検証を任せる前提です。

Enterprise 契約ユーザーでも、検証者が Enterprise owner や Organization owner 権限を持っているとは限りません。権限不足は検証の失敗ではなく、`GitHub permission limitation`、`org policy limitation`、`enterprise limitation` のどれかに分類して記録します。

### Human-only manual steps

次は人間が GitHub Web UI で実施します。コーディングエージェントに任せないでください。

- Enterprise / Organization policy の確認
- SAML / SSO / IP allow list / PAT policy の確認
- Fine-grained PAT の作成
- Fine-grained PAT request の Organization owner approval
- GHAS / Copilot 契約有無の確認
- 検証用 repository の作成と security feature 設定
- branch protection / ruleset / Actions permissions の画面設定
- screenshot を撮る場合の個人情報除去

### Agent-assisted steps

次は、手作業が完了した後にコーディングエージェントへ任せられます。

- local config の整合性確認
- `gh auth` の隔離設定確認
- `ghqr scan` の Classic PAT / FGPAT 比較実行
- `gh api` / `gh api graphql` の単体 probe
- raw results の local ignored directory への保存
- マスク済み要約の作成
- `ghqr-only failure` の有無の切り分け

### ghqr command on Linux and macOS

この runbook の `ghqr` は、検証者の shell で実行できる `ghqr` command を指します。リポジトリ内の Linux binary へ固定しません。

Linux で repo root から実行する例:

```bash
alias ghqr='./bin/linux_amd64/ghqr'
```

macOS で実行する例:

```bash
alias ghqr='./bin/darwin_arm64/ghqr'
```

または、検証者が build 済み binary を PATH に置いている場合は、そのまま `ghqr` を使います。この alias は現在の shell セッション限定にし、`.bashrc`、`.zshrc`、`.profile` などには書き込まないでください。

## 0.1 Local configuration file

公開用 template:

```text
docs/research/fgpat/fgpat-validation.example.json
```

検証者がローカルで作る private config:

```text
docs/research/fgpat/fgpat-validation.local.json
```

`fgpat-validation.local.json` は `.gitignore` 対象です。実 organization、repository、enterprise slug、output directory、masking placeholder をここに記録します。この local file は commit しません。

作成手順:

```bash
cp docs/research/fgpat/fgpat-validation.example.json docs/research/fgpat/fgpat-validation.local.json
```

必ず編集する項目:

- `targets.enterpriseSlug`
- `targets.organization`
- `targets.repositories`
- `targets.publicRepository`
- `targets.archivedRepository`
- `tokens.classicGhConfigDir`
- `tokens.fgpatGhConfigDir`
- `outputs.directory`
- `masking.*`

## 0.2 Local result storage

raw results は次の directory に保存します。

```text
docs/research/fgpat/results/
```

この directory は `.gitignore` 対象です。raw JSON / Markdown / Excel reports、command stdout/stderr logs、screen captures、API response dumps は commit しません。

公開資料に書くのは、local results から作ったマスク済み要約だけです。

## 0.3 Safe staged validation policy

Enterprise / Organization 環境では、初期検証から organization 全体を対象にした scan を実施しません。検証目的は FGPAT の互換性、endpoint ごとの permission 要件、GitHub API 制限と `ghqr` 実装差分の切り分けであり、organization 全体の inventory 収集や大規模監査ではありません。

### 初期検証ポリシー

初期検証では、organization 全 repository を対象にした scan を実施しません。

特に次の command は初期検証段階では禁止します。

```bash
ghqr scan --organization ORG_NAME
```

初期段階では、repository 明示指定 scan を優先します。

```bash
ghqr scan --repository ORG/REPO
```

この方針により、API volume を最小化し、「何を確認するためのアクセスだったか」を後から説明できる範囲に留めます。

### 推奨検証順序

対象を段階的に広げます。

1. private test repository 1 個
2. archived repository 1 個
3. 小規模 organization
4. 必要性と影響を確認してから organization scan を検討

各段階で Classic PAT と FGPAT の差分を整理し、permission 不足、GitHub API 仕様、`ghqr-only failure` のどれかに分類してから次の段階へ進みます。

### 危険度順の検証対象

初期検証では、次の順でリスクを見積もります。上にあるものほど Enterprise / Organization 管理者から見たアクセス範囲が広く、事前合意なしに実施しません。

| 危険度 | 対象 | 理由 | 初期検証での扱い |
|---|---|---|---|
| 1 | Enterprise 系 scan / discovery | Enterprise 全体の organization、audit、security settings へ広がる | 禁止。Enterprise owner と合意してから限定 probe にする。 |
| 2 | Organization 全体 scan | repository 一覧、各 repository scan、org security / actions / Copilot endpoint へ広がる | 禁止。repository 明示指定で先に切り分ける。 |
| 3 | Copilot / GHAS / Security alerts | 契約、role、security alert access log、billing access log に依存する | 必要性を整理してから endpoint 単位で確認する。 |
| 4 | Actions / Security managers | Organization administration / security role に依存する | `403` / `404` を permission limitation として記録する。 |
| 5 | archived / transferred / access 対象外 repository | FGPAT Repository access や owner 移管により visibility が変わる | local config で対象を明示し、repository 単位で確認する。 |
| 6 | Rate limit / GraphQL cost limit | repository 数や nested connection によって API 制限へ到達する | 小さい対象から始め、organization-wide query を避ける。 |

### 禁止事項

初期検証段階では、次を実施しません。

- organization 全 repository を対象にした `ghqr scan --organization ORG_NAME`
- enterprise 全体を対象にした探索的 scan
- repository 一覧を目的にした広範囲な auto-discovery
- 必要性が未整理の audit log / Copilot / security endpoint probe
- permission 不足が大量に発生する状態での繰り返し実行
- raw 結果を public docs、issue、PR、chat に貼ること

### なぜそうするのか

organization-wide scan は、限定的な repository scan よりも GitHub API へのアクセス範囲が広がります。

- organization 配下の repository 一覧列挙が発生する
- GraphQL / REST API 呼び出し量が増える
- security / actions / copilot / audit 関連 endpoint へ波及する可能性がある
- enterprise 管理者から見たときに、探索的または広範囲なアクセスに見える可能性がある
- permission 不足時の 403 / warning が大量発生する可能性がある
- GitHub audit log 上で目立つ可能性がある

そのため、最初は repository 単位で permission と endpoint 挙動を切り分けます。403 / 404 / warning の意味を整理してから、必要な範囲だけ対象を広げます。

### GitHub audit / API log 観点

Enterprise 環境では、次の観点で後から確認される可能性があります。

- GitHub audit log
- API usage monitoring
- security alert access logs
- Copilot billing access logs
- organization settings access
- code scanning / secret scanning / dependabot alert access

検証者は、各 API access について「FGPAT 互換性確認のために、限定した repository / endpoint に対して実施した」と説明できる状態を維持します。

organization-wide scan を許可する前に、次を確認します。

- repository 明示指定 scan が安定している
- permission 要件が整理されている
- audit log 上の見え方を理解している
- enterprise 管理者への説明可能性がある
- warning / 403 の整理が完了している

## 1. GitHub Enterprise prerequisites

検証前に、Enterprise owner または Organization owner が次を確認します。

### Enterprise environment

- GitHub Enterprise Cloud か GitHub Enterprise Server か
- Enterprise Managed Users (EMU) を利用しているか
- Enterprise slug を local config に記録してよいか
- Enterprise audit log へのアクセス権限を持つ管理者がいるか
- Enterprise-level API access に必要な role を持つ検証ユーザーがいるか
- 検証者が Enterprise owner ではない場合、どの項目を依頼ベースで確認するか

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

All examples assume:

- `ghqr` is available in the current shell
- target values are read from `docs/research/fgpat/fgpat-validation.local.json` by the human or agent
- raw outputs are written under `docs/research/fgpat/results/`

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
GITHUB_TOKEN="$(GH_CONFIG_DIR=/tmp/gh-classic-validation gh auth token)" ghqr scan --repository example-org/example-repo --output-name docs/research/fgpat/results/classic-repo
GITHUB_TOKEN="$(GH_CONFIG_DIR=/tmp/gh-fgpat-validation gh auth token)" ghqr scan --repository example-org/example-repo --output-name docs/research/fgpat/results/fgpat-repo
```

```bash
GITHUB_TOKEN="$(GH_CONFIG_DIR=/tmp/gh-classic-validation gh auth token)" ghqr scan --organization example-org --output-name docs/research/fgpat/results/classic-org
GITHUB_TOKEN="$(GH_CONFIG_DIR=/tmp/gh-fgpat-validation gh auth token)" ghqr scan --organization example-org --output-name docs/research/fgpat/results/fgpat-org
```

```bash
GITHUB_TOKEN="$(GH_CONFIG_DIR=/tmp/gh-classic-validation gh auth token)" ghqr scan --enterprise example-enterprise --output-name docs/research/fgpat/results/classic-enterprise
GITHUB_TOKEN="$(GH_CONFIG_DIR=/tmp/gh-fgpat-validation gh auth token)" ghqr scan --enterprise example-enterprise --output-name docs/research/fgpat/results/fgpat-enterprise
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
- whether the result is raw or masked

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
- local config file contents unless already masked

## 9. Anonymization rules

- Replace real organization names with `example-org`
- Replace real enterprise slug with `example-enterprise`
- Replace real repository names with `example-repo`
- Replace real usernames with `example-user`
- Replace internal hostnames with `example.internal`
- Replace IP addresses with `example-ip` or omit them
- Replace team names with `example-team`
- Replace email addresses with `user@example.com`
- Replace audit log actors with `example-actor`
- Replace repository IDs, node IDs, and database IDs with `example-id`
- Remove security finding details that identify internal systems
- Keep GitHub official documentation URLs as-is

## 10. Report storage

- Store raw generated reports in `docs/research/fgpat/results/`
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

## 13. Copyable prompt: manual preparation verification

Use this prompt after the human operator has completed the GitHub Web UI preparation. The agent must stop if the manual work is incomplete.

```text
You are validating whether the human-only GitHub Enterprise FGPAT preparation is complete before running ghqr.

Repository: this local ghqr checkout.
Runbook: docs/research/fgpat/enterprise-validation-runbook.md
Local config: docs/research/fgpat/fgpat-validation.local.json
Results directory: docs/research/fgpat/results/

Rules:
- Do not print token values, token fragments, prefixes, or suffixes.
- Do not read or commit GH_CONFIG_DIR contents.
- Do not commit docs/research/fgpat/results/.
- Do not commit docs/research/fgpat/fgpat-validation.local.json.
- Do not modify .bashrc, .zshrc, .profile, or other shell startup files.
- Treat ghqr as a command available in the current shell. If missing, report setup is incomplete.

Check and report:
- local config exists and is ignored by git.
- results directory is ignored by git.
- targets.enterpriseSlug, targets.organization, and targets.repositories are not placeholder values.
- masking placeholders are present.
- Classic PAT gh auth store exists or can be checked without exposing token values.
- FGPAT gh auth store exists or can be checked without exposing token values.
- Human has confirmed Enterprise Cloud or Server, EMU, SAML, PAT policy, FGPAT approval policy, IP allow list, GHAS, Copilot, Actions permissions.
- Human has confirmed FGPAT Resource owner, Repository access, Repository permissions, Organization permissions, and Organization owner approval status.
- Human has prepared the validation repository features requested by the runbook.

If any manual prerequisite is incomplete, do not run ghqr. State that manual work is incomplete and list the missing items.
If prerequisites are complete, say that validation execution can start and list the exact next commands without exposing secrets.
```

## 14. Copyable prompt: validation execution

Use this prompt only after the manual preparation verification prompt passes.

```text
Run the FGPAT Enterprise validation from docs/research/fgpat/enterprise-validation-runbook.md.

Use:
- docs/research/fgpat/fgpat-validation.local.json for targets and output locations.
- docs/research/fgpat/results/ for all raw outputs.
- ghqr command from the current shell.
- GH_CONFIG_DIR values from the local config.

Rules:
- Do not display token values, token fragments, prefixes, or suffixes.
- Do not use echo, env, or printenv to show tokens.
- Do not commit raw results, local config, GH_CONFIG_DIR contents, screenshots, or shell history.
- If ghqr is unavailable, stop and report setup is incomplete.
- If a GitHub Web UI prerequisite appears missing, stop and report manual work is incomplete.

Run paired Classic PAT and FGPAT checks for:
- repository scan
- organization scan
- auto-discovery
- enterprise discovery
- enterprise scan
- audit log endpoint
- Copilot endpoint
- security endpoints
- rulesets and branch protection
- actions permissions

For every command, record in a local results summary:
- command with secrets omitted
- token source label only
- exit code
- stdout summary
- stderr summary
- generated files
- classification: same behavior, GitHub permission limitation, org policy limitation, enterprise limitation, or ghqr-only failure
- next action if failed

After execution, create a masked summary only. Replace organization, repository, enterprise slug, usernames, team names, URLs, IPs, emails, node IDs, database IDs, and internal security details with placeholders.
```

## 15. Copyable prompt: result sanitization review

Use this prompt before sharing any validation result.

```text
Review the FGPAT validation outputs for public sharing safety.

Inputs:
- docs/research/fgpat/results/
- docs/research/fgpat/fgpat-validation.local.json
- any proposed masked summary file

Rules:
- Do not print token values, token fragments, prefixes, or suffixes.
- Do not include raw result files in commits.
- Do not include local config in commits.
- Do not include screenshots unless separately confirmed sanitized.

Verify that the proposed summary masks:
- organization names
- repository names
- enterprise slugs
- usernames
- team names
- email addresses
- internal URLs
- IP addresses
- audit actors
- repository IDs, node IDs, database IDs
- security findings that reveal internal systems

If masking is incomplete, state that the result is not safe to share and list the fields that must be fixed.
If masking is complete, state that the summary is safe to share and confirm that raw outputs remain local-only.
```
