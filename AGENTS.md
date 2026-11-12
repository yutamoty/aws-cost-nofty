# AGENTS.md

## プロジェクト概要

AWS の日次利用料金を Discord に通知する仕組み。CloudFormation 1 ファイル（`template.yaml`）で完結する。
デプロイ手順やパラメータの詳細は `README.md` を参照。

## 構成

- `template.yaml` - 全リソースの定義。Lambda のコードも `Code.ZipFile` にインラインで埋め込まれている（別ソースファイルはない）
- `README.md` - デプロイ手順、パラメータ一覧

主なリソース:

- Lambda `aws-billing-discord-notification`（Python 3.11 / arm64）: Cost Explorer（`us-east-1`）で料金を取得し、Discord Webhook に送信
- EventBridge Scheduler `aws-billing-discord-schedule`: 毎日 JST 9:00 に Lambda を起動
- EventBridge Scheduler `aws-billing-discord-test`: テスト用（1 分間隔）。通常は `DISABLED`
- Parameter Store: Discord Webhook URL を SecureString で保持（スタック外で手動登録する）

## コマンド

```bash
# テンプレートの検証（デプロイ前に必ず実行）
aws cloudformation validate-template --template-body file://template.yaml

# 初回デプロイ
aws cloudformation create-stack \
  --stack-name aws-billing-discord-notification \
  --template-body file://template.yaml \
  --capabilities CAPABILITY_NAMED_IAM

# 更新（変更セットで差分を確認してから実行する）
aws cloudformation create-change-set \
  --stack-name aws-billing-discord-notification \
  --change-set-name <name> \
  --template-body file://template.yaml \
  --capabilities CAPABILITY_NAMED_IAM
aws cloudformation describe-change-set --stack-name aws-billing-discord-notification --change-set-name <name>
```

## 開発ルール

- **デプロイ（`create-stack` / `update-stack` / `execute-change-set`）や AWS リソースの変更は、ユーザーの明示的な依頼があるときだけ行う**。
  変更セットの作成と `describe-change-set` による差分確認までは、確認用として実行してよい
- Discord Webhook URL は秘密情報。コード、テンプレート、コミット、ログ、出力に書かない。Parameter Store（SecureString）からのみ取得する
- `.env` や `*.env.*` は gitignore 済み。これらをコミットしない
- Lambda のコードを直すときは `template.yaml` の `ZipFile` 内を編集する。YAML のインデントが崩れるとコードが壊れるので、インデントを保つ
- `ZipFile` の Lambda コードは `boto3` / `urllib3` など Lambda ランタイムに同梱のものだけを使う（外部パッケージは入れられない）
- IAM ロール名などは固定名（`RoleName` を指定）。名前を変える場合は、既存リソースの置き換えが発生するので影響を確認する
- CloudFormation でリソースの Type を変えるときは、論理 ID も変更する（変更セットで型変更が通らないため。過去にスケジュールの論理 ID を変更した例がある）
- テスト用スケジュール `aws-billing-discord-test` を `ENABLED` にしたら、確認後に必ず `DISABLED` に戻す。放置すると 1 分ごとに通知が飛ぶ

## 仕様上の注意

- Cost Explorer API は `Start < End` が必須。毎月 1 日は当月の範囲が成立しないので、前月の累計にフォールバックしている（この分岐を壊さない）
- 料金は `BlendedCost` で取得。昨日の料金が $50 を超えると赤色のアラートになる
- 日付の判定は Lambda の実行時刻（UTC）の `datetime.now()` を基準にしている。タイムゾーン周りを変更するときは、JST 9:00 実行時に「昨日」が意図した日になるかを確認する
- 変更後は、実機での確認にテスト用スケジュールを使うか、Lambda を手動で 1 回実行して Discord への通知を確かめる
