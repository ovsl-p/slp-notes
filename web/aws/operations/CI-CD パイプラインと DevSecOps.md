---
title: CI/CD パイプラインと DevSecOps
aliases: [CI/CD パイプラインと DevSecOps, CodePipeline, CodeBuild, CodeDeploy, Blue/Green, Canary, ローリングデプロイ, コード署名, DevSecOps]
domain: web
tags: [aws, cicd, devsecops, deployment, codedeploy, sap-c03]
created: 2026-11-25
updated: 2026-11-26
status: growing
prerequisites: [コンテナの基本, IAM の基本]
related: ["[[IaC でインフラを揃える仕組み]]", "[[マルチアカウントのガバナンス]]", "[[AI サービスの使い分け]]", "[[責任ある AI の統制]]"]
sources:
  - https://docs.aws.amazon.com/codepipeline/latest/userguide/welcome.md
  - https://docs.aws.amazon.com/codepipeline/latest/userguide/pipelines-create-cross-account.md
  - https://docs.aws.amazon.com/codebuild/latest/userguide/welcome.md
  - https://docs.aws.amazon.com/codedeploy/latest/userguide/welcome.md
  - https://docs.aws.amazon.com/codedeploy/latest/userguide/deployment-configurations.md
  - https://docs.aws.amazon.com/codecommit/latest/userguide/history.md
  - https://docs.aws.amazon.com/AmazonECR/latest/userguide/image-signing.html
  - https://docs.aws.amazon.com/AmazonECR/latest/userguide/image-signing-verification.html
  - https://docs.aws.amazon.com/codepipeline/latest/userguide/concepts-how-it-works-conditions.html
  - https://docs.aws.amazon.com/codedeploy/latest/userguide/monitoring-cloudwatch.md
  - https://docs.aws.amazon.com/codedeploy/latest/userguide/deployment-groups-configure-advanced-options.md
  - https://docs.aws.amazon.com/AmazonECR/latest/userguide/image-scanning.md
---

# CI/CD パイプラインと DevSecOps

## ひとことで

CodePipeline が流れを管理し、CodeBuild が成果物をビルドし、CodeDeploy が環境へ配布する。検査と署名は途中に挟み、ロールバックは設定とアラームで決める。

## なぜ効くか / どこで使うか

CodePipeline、CodeBuild、CodeDeploy は役割が分かれている。どの段階で何が行われるかを押さえると、構成の抜けを見つけやすい。

使う場面は次のとおりである。

- リリース手順を自動化し、人手のミスを減らしたいとき
- 本番を止めずに新バージョンへ入れ替えたいとき
- 脆弱なイメージや未署名のコードを本番に出したくないとき
- 開発用と本番用でアカウントを分けて運用するとき

## 前提となる知識

| 用語 | 一言の説明 |
|---|---|
| ステージ | パイプラインを区切る段。複数のアクションを持つ。 |
| アクション | ステージ内の1作業。ソース取得、ビルド、デプロイなど。 |
| アーティファクト | アクション間で受け渡す成果物。S3 バケットに置かれる。 |
| デプロイ設定 | CodeDeploy が、何をどの順で入れ替えるかを定める設定。 |
| Blue/Green | 新環境を別に作り、トラフィックだけ切り替える方式。 |
| Canary | 一部のトラフィックだけ、一定時間新バージョンへ流す方式。 |
| Linear | 一定間隔ごとに、新バージョンへの割合を段階的に増やす方式。 |
| Notation | コンテナイメージに署名するためのツール。 |

## 本体の解説

### 1. CodePipeline / CodeBuild / CodeDeploy の役割分担

CodePipeline は流れの管理役である。リリース手順をステージとアクションでモデル化し、自動で回す。アーティファクトは S3 バケットに置かれ、ステージ間で渡される。ステージ条件を使うと、失敗したときの振る舞いも制御できる。

CodeBuild はビルドの実行役である。フルマネージドで、ソースのコンパイル、単体テスト、デプロイ可能な成果物の作成を行う。言語別のビルド環境が用意されているため、自前のビルドサーバは要らない。必要に応じて自動でスケールする。オンデマンドでは、使ったビルド時間だけ料金がかかる。リザーブドキャパシティは別の課金になる。

CodeDeploy は配布の実行役である。EC2・オンプレミス、Lambda、ECS へのデプロイを自動化する。入れ替えの順序や速度は、デプロイ設定で決まる。

ソースの置き場は CodeCommit に限らない。GitHub などを CodeConnections でつなげられる。CodeCommit は 2024年7月25日に新規の受付を終了したが、2025年11月25日に再開した。既存の利用者は、そのまま使える。

### 2. デプロイ戦略とロールバック

EC2/オンプレミスでは、minimum healthy hosts で速度が決まる。これは、最低限健全に保つ台数または割合である。デプロイ設定はプラットフォームごとに用意されていて、指定しなければ EC2/オンプレミスでは OneAtATime が使われる。

| 設定 | 対象 |
|---|---|
| AllAtOnce | EC2/オンプレミス、Lambda、ECS |
| HalfAtATime | EC2/オンプレミス |
| OneAtATime | EC2/オンプレミス |
| Canary | Lambda、ECS |
| Linear | Lambda、ECS |

Canary は、最初に少量だけを新バージョンに流し、待ってから残りを切り替える方式である。Linear は、一定の間隔で少しずつ流す割合を増やす方式である。割合と間隔は、Canary10Percent5Minutes や Linear10PercentEvery1Minute のような定義済みの設定から選び、カスタムもできる。

Blue/Green は、新バージョンを別環境に作ってからトラフィックを切り替える。問題が出たときは、トラフィックを戻すだけで切り戻せる。CodeDeploy は EC2、ECS、Lambda でこの方式を扱える。オンプレミスのインスタンスは Blue/Green に対応せず、インプレースのデプロイだけになる。

デプロイは CloudWatch アラームと紐づけられる。アラームがしきい値を超えると、デプロイは停止する。デプロイグループで自動ロールバックを有効にしていれば、最後に正常だったリビジョンを新しいデプロイとして配り直す。

### 3. パイプラインにセキュリティ検査と署名を差し込む

検査には Amazon Inspector を使う。ECR のイメージと Lambda 関数の脆弱性を、継続的にスキャンできる。ECR にもイメージスキャン機能が備わっている。

署名には AWS Signer を使う。ECR には署名の方法が2つある。マネージド署名は、署名設定と署名プロファイルを用意しておくと、プッシュ時に自動で署名される。手動署名は、レジストリへプッシュしたイメージを、ダイジェストを指定して Notation CLI と Signer のプラグインで署名する。Lambda のコード署名にも、AWS Signer が使える。

検証は、動かす場所によって差し込む先が変わる。EKS は admission で、Gatekeeper と Ratify の組み合わせか Kyverno を使う。ECS は、CodeDeploy なら AppSpec のフック、ECS ネイティブの blue/green ならサービスのライフサイクルフックから Lambda を呼び、本番のトラフィックを切り替える前に確かめる。手元で確かめるなら Notation CLI を使う。どちらの場合も、あらかじめ信頼ストアとトラストポリシーを設定しておく。

パイプラインの順序は、ビルド、検査、署名、配布である。検査を通ったものだけに署名し、デプロイ時に署名を検証すれば、未検査や未署名の成果物を本番の手前で止められる。

### 4. 止まったときの扱い（ステージ条件）

ステージ条件は、どのタイミングで何を検査し、失敗したらどうするかを決める。置ける場所は3つある。

| 条件 | 走るタイミング | 条件を満たさなかったときの結果 |
|---|---|---|
| Entry | ステージに入る前 | Fail か Skip |
| On Failure | ステージが失敗したとき | Rollback |
| On Success | ステージが成功したあと | Rollback か Fail |

ルールには LambdaInvoke と CloudWatchAlarm のほか、Commands、DeploymentWindow、VariableCheck などがある。CloudWatchAlarm のルールは、アラームがしきい値の内側にあれば通過する。On Success にアラームのルールを置いておくと、デプロイ直後に異常が出たときに Rollback まで進められる。

### 5. 複数アカウント・複数リージョンのパイプライン

クロスアカウントのパイプラインでは、成果物の受け渡しに KMS キーと IAM ロールの設定が必要になる。成果物を持つ S3 バケットのバケットポリシーで配布先アカウントを許可し、KMS キーはキーポリシーと配布先の IAM ポリシーの両方で復号を許可する。ロールは、信頼ポリシー（引き受けられる配布先のロール）と、パイプラインのロールの sts:AssumeRole の許可を、アカウントごとに書く。

アクションが入力として受け取れる成果物は、次のどちらかに限られる。

- パイプラインのアカウントにある成果物
- 同じアカウント内の前のアクションが作った成果物

パイプラインのアカウントを経由せずに、別の2アカウント間で直接渡すことはできない。

アカウント構成の考え方は [[マルチアカウントのガバナンス]] に、環境をコードで揃える方法は [[IaC でインフラを揃える仕組み]] にある。

## 具体例・コード

パイプラインの並びは次のとおりである。

```text
Source   : GitHub（CodeConnections 経由）
Build    : CodeBuild（コンパイル、単体テスト、イメージ作成）
Scan     : Inspector / ECR イメージスキャン
Sign     : AWS Signer（Notation でイメージに署名）
Deploy   : CodeDeploy（ECS で Canary）。署名の検証は、CodeDeploy の AppSpec のフック（AfterAllowTestTraffic）から呼ぶ Lambda で行う
Rollback : CloudWatch アラームで停止し、自動ロールバックを有効にしていれば戻す
```

本番アカウントへ配る場合は、パイプラインのアカウントが成果物を持つ。S3 のバケットポリシー、KMS のキーポリシーと配布先の IAM ポリシー、AssumeRole の両側を設定し、配布先のアクションへ渡す。

AI/ML のアプリを評価する工程を挟むなら、Bedrock のモデル評価を使う。詳細は [[AI サービスの使い分け]] にある。

## つまずきポイント

- 「CodePipeline がビルドもデプロイも実行する」は間違い。CodePipeline は流れを管理する役で、ビルドは CodeBuild、デプロイは CodeDeploy が担う。
- 「HalfAtATime や OneAtATime は Lambda でも選べる」は間違い。この2つは EC2/オンプレミス向けで、Lambda と ECS では AllAtOnce、Canary、Linear から選ぶ。
- 「デプロイ設定を指定しなければ AllAtOnce になる」は間違い。EC2/オンプレミスでは、指定がなければ OneAtATime が使われる。
- 「別の2アカウント間で成果物を直接渡せる」は間違い。成果物はパイプラインのアカウントを経由する必要があり、KMS キーと IAM ロールの設定も要る。
- 「ECR のイメージスキャンがあれば、Inspector は不要」は間違い。基本スキャンはプッシュ時か手動での都度実行で、継続的には見ない。継続的に見るなら拡張スキャン（中身は Inspector）か Inspector を使う。Lambda 関数も Inspector の対象になる。
- 「CodeCommit は新規利用できない」は間違い。2024年7月25日に新規受付を終了したが、2025年11月25日に新規の提供を再開している。

## 疑問・TODO

- [ ] ステージ条件の Skip を使うと、後続ステージの成果物がどうなるかを確認する

## 関連ノート

- [[IaC でインフラを揃える仕組み]] — パイプラインから流すインフラ側の作り方
- [[マルチアカウントのガバナンス]] — 本番と開発を分けるアカウント設計
- [[AI サービスの使い分け]] — Bedrock のモデル評価と、AI を組み込む位置
- [[責任ある AI の統制]] — AI の出力を検証・統制する層

## 出典

- https://docs.aws.amazon.com/codepipeline/latest/userguide/welcome.md
- https://docs.aws.amazon.com/codepipeline/latest/userguide/pipelines-create-cross-account.md
- https://docs.aws.amazon.com/codebuild/latest/userguide/welcome.md
- https://docs.aws.amazon.com/codedeploy/latest/userguide/welcome.md
- https://docs.aws.amazon.com/codedeploy/latest/userguide/deployment-configurations.md
- https://docs.aws.amazon.com/codecommit/latest/userguide/history.md
- https://docs.aws.amazon.com/AmazonECR/latest/userguide/image-signing.html
- https://docs.aws.amazon.com/AmazonECR/latest/userguide/image-signing-verification.html
- https://docs.aws.amazon.com/codepipeline/latest/userguide/concepts-how-it-works-conditions.html
- https://docs.aws.amazon.com/codedeploy/latest/userguide/monitoring-cloudwatch.md
- https://docs.aws.amazon.com/codedeploy/latest/userguide/deployment-groups-configure-advanced-options.md
- https://docs.aws.amazon.com/AmazonECR/latest/userguide/image-scanning.md
