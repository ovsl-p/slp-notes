# AWS（SAP-C03 の勉強から出てきた仕組み）

試験対策の判断は別リポジトリ（AWS-SAP）に置き、ここには**仕組みの理解**だけを書く。

- [[VPC の前提知識]] — IP・CIDR・最長一致・プライベート IP・NAT・DNS の土台（先に読む）
- [[AWS の VPC とネットワークの基礎]] — サブネット、ルートテーブル、NAT、エンドポイント、SG と NACL
- [[AWS の DNS と名前解決]] — Route 53 Resolver、プライベートホストゾーン、転送の向き
- [[AWS のつなぎ方の選択]] — ピアリング、Transit Gateway、Direct Connect、VPN、PrivateLink
- [[AWS のロードバランサの使い分け]] — ALB、NLB、GWLB の層と用途
- [[AWS のマルチアカウントのガバナンス]] — Organizations、SCP、Control Tower、Identity Center
- [[AWS の IAM と権限の仕組み]] — ポリシーの評価順、ロールと AssumeRole、境界・SCP・セッション、クロスアカウント
- [[AWS のストレージサービスの違い]] — S3・EBS・EFS・FSx・Storage Gateway を見せ方（オブジェクト／ブロック／ファイル）で分ける
- [[AWS のネットワークサービスの違い]] — CloudFront・Global Accelerator・Route 53 の振り分け、Transit Gateway・Direct Connect・VPN・PrivateLink の使い分け
- [[AWS のデータベースサービスの違い]] — RDS・Aurora・DynamoDB・ElastiCache・Redshift を形と目的で分ける
- [[AWS の疎結合の部品の違い]] — SQS・SNS・EventBridge・Step Functions・Kinesis を「溜める・配る・振り分ける・順に進める・流す」で分ける
- [[AWS のコンピューティングサービスの違い]] — EC2・Auto Scaling・Lambda・ECS/EKS・Fargate・Beanstalk を管理の範囲と処理の時間で分ける
- [[AWS の認証と認可の入口]] — 人・アプリ・利用者の入口（Identity Center、ロール、Cognito、Directory Service）を分ける
