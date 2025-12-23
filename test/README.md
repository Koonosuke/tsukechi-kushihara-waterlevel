# AWS 構成

Copilot 配下の現在の AWS リソース構成の俯瞰をまとめた。manifest/addons 情報に基づくため、環境差分や今後の変更は適宜反映すること。

## 環境とネットワーク

- develop / customer: 既存 VPC をインポート（develop: `vpc-01e2cc9c6c0ec2e19`、customer: `vpc-05953f861d6ff363d`）して稼働。
- staging / production: Copilot 管理の VPC（サブネットは private 配置、NAT 経由で外部通信）。
- 環境共通アドオン（stg/prod 適用）: RDS Aurora(MySQL) + Proxy、Valkey(ElastiCache Serverless)、DynamoDB（Ec_Product/Ec_Order/Outbox_Ec_Order）、S3 プライベートバケット + CloudFront、SES/CloudWatch/Chatbot/VPN/NAT 等。静的公開配信(CloudFront → S3)は production のみ。

## 構成図

```mermaid
flowchart TB
  user[利用者/外部ブラウザ]
  cf_public[CloudFront static.mycalinks.io\n(prodのみ)]
  s3_public[S3 公開コンテンツ]
  user -->|静的配信| cf_public --> s3_public

  user -->|HTTPS| waf[WAF\n(pos-web-app)]
  waf --> alb[ALB (Copilot)]

  subgraph VPC["VPC (各環境の private subnets + NAT)"]
    alb --> pos[pos-web-app\nECS Fargate + nginx]
    alb --> ecweb[ec-web-app\nECS Fargate + nginx]
    backend[backend-outbox\nBackend Service]
    pos --> cache[ElastiCache Serverless (Valkey)]
    pos --> cf_private[CloudFront private-static\n(stg/prod)] --> s3_private[S3 プライベートバケット]
    pos --> dbproxy[RDS Proxy] --> aurora[Aurora MySQL\n(stg/prod)]
    pos --> dynamo[DynamoDB\nEc_Product/Ec_Order/Outbox_Ec_Order]

    pos --> sns_pos[SNS FIFO topics\nitem/product/transaction/\nec-order/outbox/notification]
    sns_pos --> sqs_item[SQS -> worker-item] --> wi[worker-item]
    sns_pos --> sqs_product[SQS -> worker-product] --> wp[worker-product]
    sns_pos --> sqs_tx[SQS -> worker-transaction] --> wt[worker-transaction]
    sns_pos --> sqs_ecorder[SQS -> worker-ec-order] --> wec[worker-ec-order]
    sns_pos --> sqs_notif[SQS -> worker-notification] --> wn[worker-notification]

    backend --> sns_ext[SNS external-ec] --> sqs_ext[SQS -> worker-external-ec] --> wext[worker-external-ec]

    evb[EventBridge Schedules\n(stg/prod)] --> sqs_sched[SQS (worker-scheduled queue)] --> wsched[worker-scheduled]

    jobs[Scheduled Jobs\njob-daily-calculate\njob-ensure-consistency\njob-temporary-task] --> fargate_tasks[Fargate 実行\n( cron/手動 )]

    wi & wp & wt & wec & wn & wext & wsched --> dbproxy
    wi & wp & wt & wec & wn & wext & wsched --> dynamo
  end
```

## コンポーネント補足

- Web 系: `pos-web-app` と `ec-web-app` は ALB 配下の LB Web Service。`pos-web-app` に WAF を付与し、RDS Proxy/Valkey/CloudFront+S3（private）を利用。
- 非同期処理: `pos-web-app` から SNS FIFO トピック（item/product/transaction/ec-order/outbox/notification）へ publish、各トピックに紐づく SQS キュー経由で `worker-item`/`worker-product`/`worker-transaction`/`worker-ec-order`/`worker-notification` が処理。`backend-outbox` からは `external-ec` トピック経由で `worker-external-ec` が処理。
- スケジュール: EventBridge ルール（stg/prod）の出力を `worker-scheduled` キューに流し、定期タスクを実行。別途 `job-daily-calculate` / `job-ensure-consistency` / `job-temporary-task` の Scheduled Job も cron で Fargate 実行。
- データ/ストレージ: Aurora MySQL（Proxy + Route53 CNAME）、Valkey、DynamoDB 3 テーブル、S3 プライベート配信（stg/prod）と公開静的配信（prod のみ）。
- ネットワーク/運用: 各環境の VPC は private サブネット＋ NAT。CloudWatch/SES/Chatbot/VPN などのアドオンは stg/prod で有効。
