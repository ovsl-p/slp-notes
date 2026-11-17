---
title: AWS の検知と監査の違い
aliases: [AWS の検知と監査の違い, CloudTrail, Config, GuardDuty, Security Hub, Macie, Inspector, Detective, 監査ログ, 設定の評価, 脅威検知, 脆弱性検査]
domain: web
tags: [aws, cloudtrail, config, guardduty, security-hub, macie, inspector, detection, audit, sap-c03]
created: 2026-11-17
updated: 2026-11-17
status: growing
prerequisites: [API の操作（AWS への操作はすべて API 呼び出しであること）, ログと設定の違い, 検知（攻撃らしい動きを見つけること）と記録（あとから追うこと）の違い]
related: ["[[AWS の組織の統制サービスの違い]]", "[[AWS のマルチアカウントのガバナンス]]", "[[AWS の通信とアプリの防御]]", "[[AWS の暗号化と秘密情報の管理]]"]
sources:
  - https://docs.aws.amazon.com/decision-guides/latest/decision-guides/choosing-aws-security-services.html
  - https://docs.aws.amazon.com/wellarchitected/latest/management-and-governance-guide/aws-controls-and-guardrails-services.html
  - https://docs.aws.amazon.com/wellarchitected/latest/financial-services-industry-lens/key-aws-services.html
---

# AWS の検知と監査の違い

## ひとことで

AWS の検知と監査のサービスは、誰が何をしたかを記録するもの、設定が基準に合うかを見るもの、攻撃らしい動きを見つけるものに分かれる。どれも止めはしない。止めるのは予防の仕組みの役割である。

## なぜ効くか / どこで使うか

試験では「何を問われているか」を先に見極めることが要になる。「誰が API を呼んだかを追いたい」なら CloudTrail、「設定が社内基準から外れていないか」なら Config、「不正な侵入の兆候を見つけたい」なら GuardDuty が答えになりやすい。問題文の動詞（記録する、評価する、検知する、集約する）に注目すると選びやすい。逆に「止める」「拒否する」と書かれていたら、このノートのサービスではなく SCP や IAM の話になる。

## 前提となる知識

| 用語 | 一言の説明 |
|---|---|
| API | AWS への操作はすべて API 呼び出しとして行われる。コンソールも CLI も裏側は同じ API である |
| 証跡（ログ） | API 呼び出しや通信などの「起きたこと」の記録。あとから追うために残す |
| 設定 | リソースの状態（暗号化の有無、公開の範囲など）。「今どうなっているか」を表す |
| 脆弱性 | ソフトウェアやイメージに含まれる既知の弱点。修正版があるかどうかが問題になる |
| 検知と記録 | 検知は攻撃らしい動きを見つけること。記録はあとから追えるように残すこと。役割が違う |

## 本体の解説

### 1. 追うもので分ける

| どの問い | サービス | 一言で |
|---|---|---|
| 誰が何をしたか | CloudTrail | API の操作履歴を残す |
| 設定が基準に合うか | Config | 設定の変更を記録し、ルールと照らして評価する |
| 攻撃らしい動きか | GuardDuty | 継続的に監視して悪意のある活動を検知する |
| 全部を一画面に | Security Hub | 各サービスの検出結果を集めて優先度を付ける |
| S3 の中の機密 | Macie | S3 の中の個人情報などの機密データを見つける |
| ソフトの穴 | Inspector | EC2 やコンテナイメージの脆弱性を検査する |

### 2. CloudTrail（API の操作記録）

CloudTrail は、ユーザー、ロール、AWS サービスが行った操作（API 呼び出し）を記録する。監査の証跡として使うのが基本の役割である。記録するだけで、攻撃を止めることはしない。組織の証跡を設定すると、組織内の全アカウントのログを一か所に集められる。ここでは「記録の仕組み」だけを扱い、組織全体の設計は「AWS の組織の統制サービスの違い」と「AWS のマルチアカウントのガバナンス」に任せる。

### 3. Config（設定の履歴と評価）

Config は、AWS リソースの設定を記録し、設定の変更履歴を残す。さらにルール（基準）と照らして準拠しているかを評価する。ルールを束ねた適合パック（conformance pack）を使えば、まとめて適用できる。Config は設定を止めるのではなく、基準から外れていることを検出して報告するサービスである。

### 4. GuardDuty（脅威の検知）

GuardDuty は、継続的に監視して悪意のある活動を検知する。入力には CloudTrail のログ、VPC フローログ、DNS ログなどを使う。機械学習と脅威インテリジェンスを組み合わせて判断する。検知するのが役割であり、自動で遮断はしない。遮断は別の仕組みで行う。

### 5. Security Hub（検出結果の集約と基準のチェック）

Security Hub は、GuardDuty、Inspector、Macie、Config などの検出結果（findings）を一か所に集め、優先度を付ける。セキュリティ基準ごとの適合状況も一画面で見られる。Security Hub 自身も基準のチェックを行うが、脅威を見つけるのは GuardDuty の役割であり、Security Hub の主な役割は集約と要約である。

### 6. Macie（S3 の機密データの発見）と Inspector（脆弱性検査）

Macie は、S3 の中の機密データ（個人情報など）を機械学習とパターンマッチングで見つける。公開されたバケットや暗号化されていないバケットがあれば知らせる。対象は S3 に限られる。

Inspector は、EC2 やコンテナイメージ（ECR）のソフトウェアの脆弱性を継続的に検査する。ネットワークの公開範囲の検査も行う。Macie が「データの中身」を見るのに対し、Inspector は「ソフトの穴」を見る。

Detective は、検知結果を調べて原因を追うためのサービスである。このノートでは名前だけを述べ、詳細は扱わない。

## 具体例（「不審なログインがあったとき」の流れ）

1. 誰かが通常と違う操作を行うと、その操作は CloudTrail に API 呼び出しとして記録される。CloudTrail は「誰が、いつ、何をしたか」を残すだけで、そのログインを止めることはしない。
2. GuardDuty は、CloudTrail などのログを継続的に分析し、通常と違う挙動を悪意のある活動の兆候として検知し、findings を出す。遮断は行わない。
3. Security Hub は、GuardDuty の findings を他のサービスの結果と一緒に集め、優先度を付けて一画面に表示する。調べる順番を決める材料になる。
4. 次に Config で、そのユーザーが触ったリソースの設定が基準に合っているかを確認する。設定が基準から外れていれば、Config が報告する。
5. S3 の中のデータが対象になっていた場合は Macie、ソフトの穴が疑われる場合は Inspector が、それぞれ別の観点から確認に使われる。

この流れで、記録（CloudTrail）、検知（GuardDuty）、集約（Security Hub）、基準との照合（Config）の役割がそれぞれ分かれていることが見えてくる。

## つまずきポイント

- CloudTrail を入れれば攻撃が防げると思う。実際は、CloudTrail は記録するだけであり、攻撃を止めることはしない。記録は事後の追跡に使うものである。
- GuardDuty を入れれば設定の不備も見つかると思う。実際は、GuardDuty が見つけるのは攻撃らしい動き（脅威）である。設定が基準に合っているかを見るのは Config の役割である。
- Security Hub が脅威を見つけると思う。実際は、脅威を見つけるのは GuardDuty で、Security Hub は検出結果の集約と基準の適合状況の表示が主な役割である。
- Macie と Inspector は同じものだと思う。実際は、Macie は S3 の中の機密データを、Inspector はソフトウェアやイメージの脆弱性を見る。対象も目的も違う。
- 検知と記録を予防と同じものだと思う。実際は、このノートのサービスは検知と記録の役割であり、止める（予防する）役割は SCP や IAM が担う。

## 疑問・TODO

- [ ] CloudTrail のイベント履歴の保持期間を一次資料で確認する
- [ ] Config の自動修復（検知した不備を自動で直す仕組み）の有無と範囲を一次資料で確認する
- [ ] GuardDuty の料金の仕組みを一次資料で確認する

## 関連ノート

- [[AWS の組織の統制サービスの違い]] — 予防と統制の役割を整理する
- [[AWS のマルチアカウントのガバナンス]] — 複数アカウントでの設計を扱う
- [[AWS の通信とアプリの防御]] — ネットワークとアプリ側の防御を扱う
- [[AWS の暗号化と秘密情報の管理]] — データと鍵の保護を扱う

## 出典

- https://docs.aws.amazon.com/decision-guides/latest/decision-guides/choosing-aws-security-services.html
- https://docs.aws.amazon.com/wellarchitected/latest/management-and-governance-guide/aws-controls-and-guardrails-services.html
- https://docs.aws.amazon.com/wellarchitected/latest/financial-services-industry-lens/key-aws-services.html
