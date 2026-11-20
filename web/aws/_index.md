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
- [[AWS の暗号化と秘密情報の管理]] — KMS と CloudHSM（鍵を誰が持つか）、Secrets Manager とパラメータストア、ACM
- [[AWS の通信とアプリの防御]] — WAF・Shield（DDoS）・Network Firewall・セキュリティグループと NACL を、守る層で分ける
- [[AWS の組織の統制サービスの違い]] — SCP（ID の上限）・RCP（リソースの上限）・RAM（共有）・Control Tower（土台）を、何に効くかで分ける
- [[AWS の検知と監査の違い]] — CloudTrail（記録）・Config（設定の評価）・GuardDuty（脅威の検知）・Security Hub（集約）を、問いで分ける
- [[AWS の監査証拠の提出]] — Artifact（AWS 基盤の証拠）と Audit Manager（自分の利用の証拠）を、証拠の出どころで分ける
- [[AWS の AI サービスの使い分け]] — Bedrock・SageMaker AI・組み込み AI・ナレッジベース（RAG）・AgentCore を、何を選ぶ問題かで分ける
- [[AWS の責任ある AI の統制]] — Guardrails（入出力の内容）・AgentCore Identity（実行権限）・Step Functions（人の承認）を、どこで止めるかで分ける
- [[AWS のコストを下げる購入とサイジング]] — オンデマンド・Savings Plans・RI・Spot を、そのコンピュートをいつまで確実に使うかで分ける
- [[AWS のコストの配分と可視化]] — タグ・Cost Categories・CUR 2.0・Budgets で、誰の支出かを説明できる形にする
