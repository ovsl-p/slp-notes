---
title: CI/CD パイプラインと DevSecOps
aliases: [CI/CD パイプラインと DevSecOps, CodePipeline, CodeBuild, CodeDeploy, Blue/Green, Canary, ローリングデプロイ, コード署名, DevSecOps]
domain: web
tags: [aws, cicd, devsecops, deployment, codedeploy, sap-c03]
created: 2026-11-25
updated: 2026-11-25
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
---

# CI/CD パイプラインと DevSecOps

## ひとことで

CodePipeline が流れを組み、CodeBuild が作り、CodeDeploy が配る。検査と署名は途中に挟み、戻し方はデプロイ設定とアラームで決める。

## なぜ効くか / どこで使うか

3つのサービスは仕事が分かれている。どこで何が起きるかを押さえると、構成の穴がどこにあるかを見つけやすい。

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

CodeBuild はビルドの実行役である。フルマネージドで、ソースのコンパイル、単体テスト、デプロイ可能な成果物の作成を行う。言語別のビルド環境が用意されているため、自前のビルドサーバは要らない。必要に応じて自動でスケールし、使ったビルド時間だけ料金がかかる。

CodeDeploy は配布の実行役である。EC2・オンプレミス、Lambda、ECS へのデプロイを自動化する。入れ替えの順序や速度は、デプロイ設定で決まる。

ソースの置き場は CodeCommit に限らない。GitHub などを CodeConnections でつなげられる。CodeCommit は 2024年7月25日に新規の受付を終了した。その後、2025年11月25日に新規の提供を再開している。既存の利用者は、そのまま使える。

### 2. デプロイ戦略とロールバック

EC2/オンプレミスでは、minimum healthy hosts で速度が決まる。これは、最低限健全に保つ台数または割合である。設定はプラットフォームごとに用意されていて、指定しなければ OneAtATime が使われる。

| 設定 | 対象 |
|---|---|
| AllAtOnce | EC2/オンプレミス、Lambda、ECS |
| HalfAtATime | EC2/オンプレミス |
| OneAtATime | EC2/オンプレミス |
| Canary | Lambda、ECS |
| Linear | Lambda、ECS |

Canary の例は、10%を5分間だけ新バージョンに流す方式である。Linear の例は、1分ごとに10%ずつ増やす方式である。

Blue/Green は、新バージョンを別環境に作ってからトラフィックを切り替える。問題が出たときは、トラフィックを戻すだけで切り戻せる。CodeDeploy は EC2/オンプレミス、ECS、Lambda でこの方式を扱える。

ロールバックは、CloudWatch アラームと紐づけられる。しきい値を超えると、自動で前の状態に戻る。

### 3. パイプラインにセキュリティ検査と署名を差し込む

検査には Amazon Inspector を使う。ECR のイメージと Lambda 関数の脆弱性を、継続的にスキャンできる。ECR にもイメージスキャン機能が備わっている。

署名には AWS Signer を使う。コンテナイメージは Notation で署名し、デプロイ時に検証できる。Lambda のコード署名にも、AWS Signer が使える。

順序は、ビルド、検査、署名、配布である。デプロイのときに署名を検証すれば、未検査や未署名の成果物を本番の手前で止められる。

### 4. 複数アカウント・複数リージョンのパイプライン

クロスアカウントのパイプラインでは、成果物の受け渡しに KMS キーと IAM ロールの設定が必要になる。

受け取れる成果物には制約がある。あるアクションが受け取れるのは、次のどちらかに限られる。

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
Deploy   : CodeDeploy（ECS で Canary、デプロイ時に署名を検証）
Rollback : CloudWatch アラームで自動ロールバック
```

本番アカウントへ配る場合は、パイプラインのアカウントが成果物を持つ。KMS キーと IAM ロールを設定し、配布先のアクションへ渡す。

AI/ML のアプリを評価する工程を挟むなら、Bedrock のモデル評価を使う。詳細は [[AI サービスの使い分け]] にある。

## つまずきポイント

- 「CodePipeline がビルドもデプロイも実行する」は間違い。CodePipeline は流れを管理する役で、ビルドは CodeBuild、デプロイは CodeDeploy が担う。
- 「HalfAtATime や OneAtATime は Lambda でも選べる」は間違い。この2つは EC2/オンプレミス向けで、Lambda と ECS では AllAtOnce、Canary、Linear から選ぶ。
- 「デプロイ設定を指定しなければ AllAtOnce になる」は間違い。EC2/オンプレミスでは、指定がなければ OneAtATime が使われる。
- 「別の2アカウント間で成果物を直接渡せる」は間違い。成果物はパイプラインのアカウントを経由する必要があり、KMS キーと IAM ロールの設定も要る。
- 「ECR のイメージスキャンがあれば、Inspector は不要」は間違い。Inspector は ECR のイメージと Lambda 関数を継続的にスキャンでき、署名は別に AWS Signer で行う。
- 「CodeCommit は新規利用できない」は間違い。2024年7月25日に新規受付を終了したが、2025年11月25日に新規の提供を再開している。

## 疑問・TODO

- [ ] ステージ条件で設定できる失敗時の振る舞いを、具体的に確認する
- [ ] ECR のイメージ署名で、デプロイ時の検証を組み込む手順を確認する

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
