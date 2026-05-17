# Fine-Grained PAT research materials

この branch は、`ghqr` の Fine-grained personal access token (FGPAT) 対応を調査し、GitHub Enterprise 契約ユーザーが追加検証を引き継げるようにするための research branch です。

この資料群は upstream PR にそのまま含める前提ではありません。upstream 向けには、ここで得た結論のうち README、warning、troubleshooting に必要な最小部分だけを切り出します。

## Files

- `classic-vs-fgpat-compatibility-matrix.md`
  - Classic PAT と FGPAT の公式仕様差、用語定義、ghqr への影響、README 反映候補
- `fgpat-full-validation.md`
  - FGPAT 検証の全体設計、source code 調査、GitHub 公式仕様の整理
- `fgpat-validation-design.md`
  - 初期検証の時系列ログと判断基準
- `fgpat-pr-plan.md`
  - 改善 PR に向けた調査計画と判断材料
- `fgpat-capability-diff.md`
  - Classic PAT と FGPAT の機能差分、実測結果、現時点の最終判断
- `enterprise-validation-runbook.md`
  - Enterprise 契約ユーザー向けの追加検証手順
- `fgpat-validation.example.json`
  - ローカル検証設定の公開可能なテンプレート

## Recommended reading order

目的ごとに読む資料を分けます。全ファイルを順番に読む必要はありません。

1. 全体像と公開ルールを確認する
   - `README.md`
2. FGPAT 関連用語、Classic PAT との差分、FGPAT で詰まりやすい領域を確認する
   - `classic-vs-fgpat-compatibility-matrix.md`
3. 既存の実測結果と ghqr-only failure の有無を確認する
   - `fgpat-capability-diff.md`
4. Enterprise 契約環境で安全に追加検証する
   - `enterprise-validation-runbook.md`
5. 初期検証の時系列や token 取り扱いの経緯を確認する
   - `fgpat-validation-design.md`

資料の役割:

- `classic-vs-fgpat-compatibility-matrix.md` は、公開説明や README / troubleshooting へ転用する根拠資料です。
- `enterprise-validation-runbook.md` は、実 organization / enterprise で追加検証する人向けの手順書です。
- `fgpat-full-validation.md` と `fgpat-pr-plan.md` は、調査過程を残すための詳細ログです。

## Local-only files

検証者は、次の local-only file / directory を使います。これらは `.gitignore` 対象で、commit しません。

- `docs/research/fgpat/fgpat-validation.local.json`
  - 実 organization、repository、enterprise slug、output path、masking placeholder を記録するローカル設定
- `docs/research/fgpat/results/`
  - raw JSON / Markdown / Excel reports、stdout/stderr logs、API response dumps、screen capture などを保存するローカル結果ディレクトリ

作成例:

```bash
cp docs/research/fgpat/fgpat-validation.example.json docs/research/fgpat/fgpat-validation.local.json
mkdir -p docs/research/fgpat/results
```

## Public sharing rules

- token 値、token の一部、prefix、末尾は記録しない
- screenshot には個人名、メールアドレス、内部 URL、IP、enterprise slug、organization slug を含めない
- 実 organization 名、実 repository 名、実 username は placeholder に置換する
- `GH_CONFIG_DIR` の中身、`.env`、shell history、`.codex`、生成 report は commit しない
- この資料における `ghqr` は、検証者の shell で実行可能な `ghqr` command を指す
- Linux では `alias ghqr='./bin/linux_amd64/ghqr'` のような shell セッション限定 alias を使える
- macOS では build 済み macOS binary または PATH 上の `ghqr` を使う
- `.bashrc` などの恒久的な shell 設定は変更しない

## Current conclusion

- repository scan と organization scan は、適切に設定した FGPAT でも動作する
- enterprise / audit 系は未検証領域が残り、enterprise permission、GitHub plan、organization policy に依存する
- 現時点で REST / GraphQL 単体は成功するが `ghqr` だけ失敗する箇所は確認されていない
- FGPAT は Classic PAT の完全上位互換ではないため、README では「一部機能では同等に動くが、GitHub API の FGPAT 対応状況、permission、org approval policy により差が出る可能性がある」と記載するのが妥当

## Pre-push safety check

Before pushing this branch, confirm that the commit does not include:

- token values or token fragments
- token prefixes or suffixes
- email addresses
- real enterprise slugs
- internal URLs
- IP addresses
- real usernames
- screenshots
- generated reports
- temporary files
- shell history copies
- `.env`
- `.codex`
- `GH_CONFIG_DIR` contents

Current check result:

- Markdown research files were copied into `docs/research/fgpat/`
- Generated JSON / Markdown / Excel scan reports remain outside this docs tree and are not intended for commit
- Source-specific names were replaced with public placeholders such as `example-org`, `example-user`, and `fgpat-validation-repo`
- `fgpat-validation.local.json` and `results/` are ignored by git
- Public-safety scan completed for docs/research/fgpat before commit

## Expected upstream split

Likely appropriate for upstream PR:

- README guidance for FGPAT setup
- Troubleshooting steps for 403 / 404 / empty discovery results
- Clarification that FGPAT is not guaranteed to be fully equivalent to Classic PAT
- Optional warning improvements if the message is too generic

Likely not appropriate for upstream PR:

- raw research logs
- generated scan reports
- private organization-specific command outputs
- Enterprise-only validation details that have not been generalized
