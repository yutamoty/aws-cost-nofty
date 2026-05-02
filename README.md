# aws-cost-nofty

AWS の日次利用料金を Discord に通知する仕組みです。

## 構成

- **Lambda** (Python 3.11, arm64): Cost Explorer API で利用料金を取得し、Discord Webhook で通知
- **EventBridge Scheduler**: 毎日 JST 9:00 に Lambda を起動
- **Parameter Store**: Discord Webhook URL を SecureString で管理
- **CloudFormation**: 上記リソースを一括デプロイ

## 通知内容

- 前日の利用料金
- 前日比（増減）
- 当月の累計料金
- $50 超過時の高額アラート

## デプロイ手順

### 1. Discord Webhook URL を Parameter Store に登録

```bash
aws ssm put-parameter \
  --name "/billing-notification/discord-webhook-url" \
  --value "<YOUR_DISCORD_WEBHOOK_URL>" \
  --type "SecureString" \
  --description "Discord webhook URL for AWS billing notifications"
```

### 2. CloudFormation スタックをデプロイ

```bash
aws cloudformation create-stack \
  --stack-name aws-billing-discord-notification \
  --template-body file://template.yaml \
  --capabilities CAPABILITY_NAMED_IAM
```

### 3. (任意) テスト実行

EventBridge Scheduler のテストスケジュール `aws-billing-discord-test` を ENABLED にすると 1 分間隔で実行されます。確認後は DISABLED に戻してください。

## パラメータ

| パラメータ | デフォルト値 | 説明 |
|---|---|---|
| `DiscordWebhookParameterName` | `/billing-notification/discord-webhook-url` | Parameter Store のパラメータ名 |
| `NotificationTime` | `cron(0 9 * * ? *)` | 通知スケジュール (`ScheduleTimeZone` のタイムゾーンで解釈) |
| `ScheduleTimeZone` | `Asia/Tokyo` | スケジュールのタイムゾーン (IANA 形式) |
