---
title: AWS のコストを下げる購入とサイジング
aliases: [AWS のコストを下げる購入とサイジング, Savings Plans, リザーブドインスタンス, Reserved Instances, Spot インスタンス, 購入オプション, Compute Optimizer, コスト最適化]
domain: web
tags: [aws, cost, pricing, savings-plans, spot, compute-optimizer, sap-c03]
created: 2026-11-20
updated: 2026-11-20
status: growing
prerequisites: [コンピュートの課金単位, インスタンスのサイジング]
related: ["[[AWS のコンピューティングサービスの違い]]", "[[AWS の AI サービスの使い分け]]", "[[AWS のコストの配分と可視化]]"]
sources:
  - https://docs.aws.amazon.com/savingsplans/latest/userguide/what-is-savings-plans.html
  - https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-reserved-instances.html
  - https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-spot-instances.html
  - https://docs.aws.amazon.com/compute-optimizer/latest/ug/what-is-compute-optimizer.html
  - https://aws.amazon.com/step-functions/pricing/
  - https://docs.aws.amazon.com/AmazonECS/latest/developerguide/fargate-capacity-providers.html
  - https://aws.amazon.com/ec2/graviton/
  - https://aws.amazon.com/bedrock/pricing/
---

# AWS のコストを下げる購入とサイジング

## ひとことで

コストは、そのコンピュートをいつまで確実に使うかで購入オプションを選ぶと下がる。常時は Savings Plans、中断できる処理は Spot、読めない利用はオンデマンドで払う。

## なぜ効くか / どこで使うか

AWS の料金は、使った量と時間に対して課金される。同じ処理でも購入の仕方によって単価が変わる。割引の多くは、期間や利用量を先に約束することと引き換えに受けられる。Spot は約束ではなく、中断を受け入れる代わりに安くなる。

使うのは、EC2、Fargate、Lambda などのコンピュート費用を見直すときである。見直しの軸は、そのコンピュートを使う期間が読めるかどうかにある。

## 前提となる知識

| 用語 | 意味 |
|---|---|
| コミット | 一定期間、一定の利用額や利用量を事前に約束すること。 |
| オンデマンド | 約束なしで、使った分だけ払う基準の料金モデル。 |
| 中断 | AWS 側の都合で、Spot インスタンスなどが途中で止められること。 |
| 割引の対象範囲 | 割引がどのリソースに適用されるか。購入オプションごとに異なる。 |
| rightsizing | 利用率に合わせて、インスタンスなどのサイズを適正化すること。 |
| 状態遷移 | Step Functions のワークフローで、状態が1つ進む単位。課金の単位にもなる。 |
| 価格性能 | 単価と処理性能を合わせて見た、費用対効果。 |

## 本体の解説

### 1. 購入オプションの選び方

| 購入オプション | 約束 | 向く workload | 割引の目安 |
|---|---|---|---|
| オンデマンド | なし | 予測できない、短命な、止められる workload | 基準の単価 |
| Savings Plans | 1年または3年、時間あたりの利用額（USD/時）をコミット | 常時稼働する利用 | 最大72% |
| Reserved Instances | 特定のインスタンス構成へのコミット | 構成が固定の常時稼働 | 最大72%。AWS は Savings Plans を推奨している |
| Spot | なし。中断を受け入れる | 中断を許容できる処理（データ分析、バッチ、バックグラウンド） | オンデマンドより安い |

Savings Plans は、**コミット額**を時間あたりの金額で決める。インスタンスの台数や型を決める方式ではない。Compute Savings Plans は、EC2 のインスタンスファミリー、サイズ、OS、テナンシー、リージョンを問わず適用される。Fargate と Lambda の利用にも効く。SageMaker AI のインスタンス利用には、SageMaker AI Savings Plans が割引を適用する。

Reserved Instances は、特定のインスタンス構成へのコミットである。物理的なインスタンスを確保する仕組みではなく、請求上の割引として働く。柔軟性の面で Savings Plans が優れるため、AWS は Savings Plans の利用を推奨している。

Spot は、EC2 の余剰容量をオンデマンドより安く使う仕組みである。**中断**されることがあるため、途中で止まっても問題のない処理に使う。Spot 価格はインスタンスタイプと AZ ごとに設定され、需給によって徐々に変動する。

### 2. サイジングを先に行う（Compute Optimizer）

AWS Compute Optimizer は、構成と利用率のメトリクスを分析する。rightsizing の推奨を出し、遊休リソースも検出する。

サイズを適正化してから購入オプションを決める。過大なインスタンスに対して先にコミットすると、無駄な分の割引まで固定されてしまう。順序は、サイジング、次に購入オプションである。

### 3. ワークフローとコンテナの選択

Step Functions の Standard ワークフローは、状態遷移の数で課金される。月4,000遷移まで無料で、リトライも1遷移として数えられる。

Step Functions の Express ワークフローは、リクエスト数と実行時間で課金される。実行時間は100ms 単位に切り上げられ、メモリ使用量も料金の要素になる。

```
Standard の概算 = 1回の実行の遷移数 × 実行回数
Express の概算 = リクエスト数 + 実行時間（100ms 単位）× メモリ要素
```

どちらが安いかは、この式に実際の値を入れて比べるまで決まらない。

Fargate Spot は、中断を許容できるコンテナ処理に使う容量オプションである。ECS のキャパシティプロバイダーを通じて指定する。

Graviton は、AWS がクラウド workload に対して最も良い価格性能を出すと説明しているプロセッサ群である。同じ処理をより安い単価で実行できる選択肢として扱う。

### 4. AI/ML のコスト（バッチ推論）

Bedrock のバッチ推論は、オンデマンドより50%安い。結果をその場で返す必要がない処理に向く。

スループットを固定する Provisioned Throughput は、モデルユニット単位で払う方式になる。詳細は [[AWS の AI サービスの使い分け]] で扱う。

## 具体例・コード

ある会社のシステムを、常時稼働の API、夜間バッチ、月次のレポート生成の3つに分けて考える。

1. 3つの workload を、止められるかどうかと利用の時間帯で分ける。API は止められず、夜間バッチは中断を許容できる。月次レポートは月に一度だけ動く。
2. 各 workload の構成と利用率を Compute Optimizer で確認し、サイズを適正化する。
3. 常時稼働の API は、適正化後の利用額を見て Compute Savings Plans をコミットする。ファミリーやサイズを変えても割引が続くため、構成変更に強い。
4. 夜間バッチは Spot（EC2）または Fargate Spot で動かす。中断されたときに再実行できる設計にする。
5. 月次レポートはオンデマンドで実行する。レポートに生成 AI を使う場合は、Bedrock のバッチ推論が使えるか確認する。

月次レポートを約束で縛らないのは、実行回数が少なく、コミットの割引が無駄になりやすいからである。

## つまずきポイント

- 「RI は物理的なインスタンスを予約する仕組みである」は間違い。RI は請求上の割引であり、物理的な容量を確保するものではない。
- 「Compute Savings Plans はインスタンスタイプとリージョンを指定して買う」は間違い。ファミリー、サイズ、OS、テナンシー、リージョンを問わず適用され、Fargate と Lambda にも効く。
- 「Spot は割引が大きいので本番の API に使える」は間違い。中断されることがあるため、中断を許容できる処理に限って使う。
- 「Spot の価格は一定の割引率で固定されている」は間違い。価格はインスタンスタイプと AZ ごとに設定され、需給で徐々に変動する。
- 「Compute Optimizer が推奨どおりに構成を変更する」は間違い。分析と推奨、遊休リソースの検出を行うものであり、変更は利用者が行う。
- 「Step Functions の Standard と Express は同じ単位で課金される」は間違い。Standard は状態遷移数、Express はリクエスト数と実行時間（100ms 単位）とメモリで課金される。

## 疑問・TODO

- [ ] Savings Plans のコミット額を超えた分の課金方法を確認する
- [ ] Reserved Instances の期間と支払い方式の種類を確認する
- [ ] Graviton で動かせる条件（ランタイムとライブラリの対応）を確認する
- [ ] Compute Optimizer が推奨を出すために必要な利用データの期間を確認する

## 関連ノート

- [[AWS のコンピューティングサービスの違い]] — EC2・Fargate・Lambda のどれを選ぶかで、購入オプションの効き方が変わる
- [[AWS の AI サービスの使い分け]] — Bedrock のバッチ推論と Provisioned Throughput の位置づけ
- [[AWS のコストの配分と可視化]] — 下げた結果を誰のコストとして説明するか

## 出典

- https://docs.aws.amazon.com/savingsplans/latest/userguide/what-is-savings-plans.html
- https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-reserved-instances.html
- https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-spot-instances.html
- https://docs.aws.amazon.com/compute-optimizer/latest/ug/what-is-compute-optimizer.html
- https://aws.amazon.com/step-functions/pricing/
- https://docs.aws.amazon.com/AmazonECS/latest/developerguide/fargate-capacity-providers.html
- https://aws.amazon.com/ec2/graviton/
- https://aws.amazon.com/bedrock/pricing/
