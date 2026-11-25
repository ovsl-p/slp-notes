# AWS（SAP-C03 の勉強から出てきた仕組み）

試験対策の判断は別リポジトリ（AWS-SAP）に置き、ここには**仕組みの理解**だけを書く。フォルダはテーマで分けてある。

## network

- [[VPC の前提知識]] — IP・CIDR・最長一致・プライベート IP・NAT・DNS の土台（先に読む）
- [[VPC とネットワークの基礎]] — サブネット、ルートテーブル、NAT、エンドポイント、SG と NACL
- [[DNS と名前解決]] — Route 53 Resolver、プライベートホストゾーン、転送の向き
- [[つなぎ方の選択]] — ピアリング、Transit Gateway、Direct Connect、VPN、PrivateLink
- [[ロードバランサの使い分け]] — ALB、NLB、GWLB の層と用途
- [[ネットワークサービスの違い]] — CloudFront・Global Accelerator・Route 53 の振り分け、Transit Gateway・Direct Connect・VPN・PrivateLink の使い分け
- [[通信とアプリの防御]] — WAF・Shield（DDoS）・Network Firewall・セキュリティグループと NACL を、守る層で分ける

## security

- [[IAM と権限の仕組み]] — ポリシーの評価順、ロールと AssumeRole、境界・SCP・セッション、クロスアカウント
- [[認証と認可の入口]] — 人・アプリ・利用者の入口（Identity Center、ロール、Cognito、Directory Service）を分ける
- [[暗号化と秘密情報の管理]] — KMS と CloudHSM（鍵を誰が持つか）、Secrets Manager とパラメータストア、ACM
- [[組織の統制サービスの違い]] — SCP（ID の上限）・RCP（リソースの上限）・RAM（共有）・Control Tower（土台）を、何に効くかで分ける
- [[マルチアカウントのガバナンス]] — Organizations、SCP、Control Tower、Identity Center
- [[検知と監査の違い]] — CloudTrail（記録）・Config（設定の評価）・GuardDuty（脅威の検知）・Security Hub（集約）を、問いで分ける
- [[監査証拠の提出]] — Artifact（AWS 基盤の証拠）と Audit Manager（自分の利用の証拠）を、証拠の出どころで分ける

## compute-data

- [[データレイクと分析の部品の違い]] — S3・Glue Data Catalog・Lake Formation・Athena・Redshift・EMR・Kinesis を、置く／台帳／権限／問い合わせる／流すで分ける
- [[コンピューティングサービスの違い]] — EC2・Auto Scaling・Lambda・ECS/EKS・Fargate・Beanstalk を管理の範囲と処理の時間で分ける
- [[ストレージサービスの違い]] — S3・EBS・EFS・FSx・Storage Gateway を見せ方（オブジェクト／ブロック／ファイル）で分ける
- [[データベースサービスの違い]] — RDS・Aurora・DynamoDB・ElastiCache・Redshift を形と目的で分ける
- [[疎結合の部品の違い]] — SQS・SNS・EventBridge・Step Functions・Kinesis を「溜める・配る・振り分ける・順に進める・流す」で分ける

## ai

- [[AI サービスの使い分け]] — Bedrock・SageMaker AI・組み込み AI・ナレッジベース（RAG）・AgentCore を、何を選ぶ問題かで分ける
- [[責任ある AI の統制]] — Guardrails（入出力の内容）・AgentCore Identity（実行権限）・Step Functions（人の承認）を、どこで止めるかで分ける

## cost

- [[コストを下げる購入とサイジング]] — オンデマンド・Savings Plans・RI・Spot を、そのコンピュートをいつまで確実に使うかで分ける
- [[コストの配分と可視化]] — タグ・Cost Categories・CUR 2.0・Budgets で、誰の支出かを説明できる形にする

## resilience

- [[クラウド移行の戦略と道具]] — 7R と、MGN・DMS・DataSync・AWS Transform を、移す／残す／やめる／作り直すの判断で分ける
- [[災害対策の4つの型と切り替えの道具]] — バックアップ／パイロットライト／ウォームスタンバイ／マルチサイトを、RPO・RTO・コストで選ぶ
- [[バックアップの一元管理と保護の仕組み]] — AWS Backup の集中管理と、Vault Lock・Object Lock・スナップショットロックによる削除・改変の防止
- [[IaC でインフラを揃える仕組み]] — CloudFormation と CDK、Hooks・Aspects・Config・SCP で止める位置、StackSets での配布
- [[CI-CD パイプラインと DevSecOps]] — CodePipeline・CodeBuild・CodeDeploy の分担、デプロイ戦略、署名とスキャン、クロスアカウント
