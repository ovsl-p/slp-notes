---
title: AWS の通信とアプリの防御
aliases: [AWS の通信とアプリの防御, WAF, Shield, Network Firewall, セキュリティグループ, NACL, DDoS, Web ACL, ステートフル, ステートレス]
domain: web
tags: [aws, waf, shield, network-firewall, security-group, nacl, sap-c03]
created: 2026-11-16
updated: 2026-11-16
status: growing
prerequisites: [VPC の前提知識, ステートフルとステートレスの違い]
related: ["[[AWS の VPC とネットワークの基礎]]", "[[AWS の IAM と権限の仕組み]]", "[[AWS のロードバランサの使い分け]]"]
sources:
  - https://docs.aws.amazon.com/waf/latest/developerguide/how-aws-waf-works.html
  - https://docs.aws.amazon.com/waf/latest/developerguide/ddos-advanced-summary.html
  - https://docs.aws.amazon.com/network-firewall/latest/developerguide/what-is-aws-network-firewall.html
  - https://docs.aws.amazon.com/vpc/latest/userguide/security-group-rules.html
  - https://docs.aws.amazon.com/vpc/latest/userguide/vpc-network-acls.html
---

# AWS の通信とアプリの防御

## ひとことで

このメモでは、AWS で通信を守る5つの仕組みの役割の違いを扱います。試験では、守る対象の層で選択肢を見分けます。

## なぜ効くか / どこで使うか

通信を守る仕組みは、守る層によって使い分けます。攻撃には、Web アプリの中身を狙うものと、大量のトラフィックで対象を埋めるものがあります。層に合わない仕組みを選ぶと、攻撃を防げません。

使う場面は、公開 Web アプリの前段、VPC 内の通信の検査、リソース単位の許可設定などです。

## 前提となる知識

| 用語 | 一言の説明 |
|---|---|
| ステートフル | 通信の状態を覚え、戻りの通信を自動で判断する方式です。 |
| ステートレス | 通信の状態を覚えず、通過するパケットを個別に判断する方式です。 |
| CIDR | IP アドレスの範囲を「10.0.0.0/16」のような表記で表すものです。 |
| ポート | 同じ IP アドレスの中で、通信の行き先のサービスを分ける番号です。 |
| DDoS | 大量のリクエストやパケットで対象を過負荷にする攻撃です。 |
| VPC | AWS 上に作る、論理的に分けたプライベートなネットワークです。 |
| L3/L4、L7 | OSI 参照モデルの層番号です。L3/L4 はネットワークとトランスポート、L7 は HTTP などのアプリケーションの層です。 |

VPC の基礎は [[VPC の前提知識]] にまとめてあります。

## 本体の解説

### 1. 守る層で分ける

| 層 | サービス | 一言で |
| --- | --- | --- |
| L3/L4（ネットワーク、トランスポート） | Shield Standard、Shield Advanced | 大量のパケットによる DDoS を防ぎます。 |
| L7（HTTP の中身） | WAF | リクエストの内容を Web ACL のルールで検査します。 |
| VPC 内の通信 | Network Firewall | VPC を通る通信の中身をルールで検査します。 |
| EC2 などのリソース単位 | セキュリティグループ | ENI 単位で通信を許可します。 |
| サブネット単位 | NACL | サブネットの出入りを許可と拒否で制御します。 |

### 2. WAF

WAF は HTTP(S) のリクエストを検査するサービスです。検査には Web ACL を使います。Web ACL は、ルールを集めたものです。

ルールに一致したリクエストは、許可、遮断、カウントのいずれかで処理します。CAPTCHA やチャレンジを出すこともできます。

WAF は CloudFront、ALB、API Gateway の前に付けて使います。ルールは自作のほか、AWS マネージドルールグループ（既成の攻撃パターン）を使えます。ルールの容量は WCU という単位で管理します。

### 3. Shield

Shield Standard は、全利用者に自動で付きます。追加料金はかかりません。ネットワーク層とトランスポート層（L3/L4）の DDoS を防ぎます。

Shield Advanced は有料のサブスクリプションです。L7 の DDoS 対策を含みます。L7 の対策には、WAF のマネージドルールを使います。ほかに、DDoS 対応チームへの相談や、攻撃による料金の補償があります。

### 4. Network Firewall

Network Firewall は VPC の中の通信を検査するファイアウォールです。サブネットに置いたエンドポイントを経由させるよう、ルートテーブルで向けます。

ルールエンジンは、ステートレスとステートフルの2種類があります。ステートフルでは、ドメイン名での許可や拒否と、Suricata 形式の IPS ルールを書けます。

VPC をまたいで集中的に検査する場合は、Transit Gateway 経由の構成にします。

### 5. セキュリティグループと NACL の違い

| 項目 | セキュリティグループ | NACL |
|---|---|---|
| 適用単位 | EC2 などのリソース（ENI） | サブネット |
| 状態の扱い | ステートフル | ステートレス |
| 書けるルール | 許可のみ | 許可と拒否 |
| 戻りの通信 | 自動で許可 | 明示的に許可が必要 |
| 評価の方法 | すべてのルールを評価 | 番号の小さい順に評価し、最初に一致したもので決まる |

セキュリティグループと NACL は、両方で通る必要があります。どちらか一方でも拒否すれば、通信は止まります。

## 具体例・コード

Web アプリ1つを例に、通信の流れと防御を整理します。

1. ユーザーのリクエストは CloudFront に届きます。WAF の Web ACL を CloudFront に付け、マネージドルールで既知の攻撃を遮断します。
2. CloudFront から ALB へ転送されます。Shield Standard は、この経路の L3/L4 の DDoS を自動で防ぎます。
3. ALB のセキュリティグループは、送信元を CloudFront からのアクセスに限定します。
4. アプリの EC2 のセキュリティグループは、ALB のセキュリティグループを参照して、ALB からの通信だけを許可します。

```
app-sg  inbound: TCP 8080, source = alb-sg
```

5. プライベートサブネットの NACL は、必要な戻りの通信も含めて許可します。外部 API への外向き通信は、Network Firewall で検査します。

## つまずきポイント

- Shield Advanced を入れれば DDoS をすべて防げると思う。実際は、Advanced は有料で、L7 の対策には WAF のルールが必要です。
- WAF は EC2 に直接付けられると思う。実際は、CloudFront、ALB、API Gateway などの前に付けます。
- Network Firewall はすべての通信を自動で検査すると思う。実際は、ルートテーブルで経由させたサブネットの通信だけが対象です。
- セキュリティグループで拒否のルールを書けばよいと思う。実際は、セキュリティグループは許可のみで、拒否は NACL で書きます。
- Shield Standard は契約して申し込む必要があると思う。実際は、全利用者に自動で付き、追加料金もかかりません。

## 疑問・TODO

- [ ] WAF のマネージドルールグループの種類と、CloudFront と ALB での違いを確認する
- [ ] Network Firewall の集中検査の構成を、Transit Gateway と合わせて整理する
- [ ] Shield Advanced の料金補償の条件を公式ドキュメントで確認する

## 関連ノート

- [[AWS の VPC とネットワークの基礎]] — 通信を通す土台（ルートテーブル、サブネット）
- [[AWS の IAM と権限の仕組み]] — 誰が通信の設定を変えられるか
- [[AWS のロードバランサの使い分け]] — WAF を前に付ける ALB の選び方

## 出典

- https://docs.aws.amazon.com/waf/latest/developerguide/how-aws-waf-works.html
- https://docs.aws.amazon.com/waf/latest/developerguide/ddos-advanced-summary.html
- https://docs.aws.amazon.com/network-firewall/latest/developerguide/what-is-aws-network-firewall.html
- https://docs.aws.amazon.com/vpc/latest/userguide/security-group-rules.html
- https://docs.aws.amazon.com/vpc/latest/userguide/vpc-network-acls.html
