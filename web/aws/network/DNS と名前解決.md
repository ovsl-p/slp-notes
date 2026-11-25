---
title: DNS と名前解決
aliases: [DNS と名前解決, Route 53 Resolver, プライベートホストゾーン, 名前解決, DNS 転送, Resolver エンドポイント]
domain: web
tags: [aws, dns, route53, resolver, hybrid, private-hosted-zone]
created: 2026-11-14
updated: 2026-11-25
status: growing
prerequisites: [DNS の仕組み, VPC の基礎, 再帰問い合わせと転送, パブリック DNS とプライベート DNS の違い]
related: ["[[VPC とネットワークの基礎]]", "[[つなぎ方の選択]]", "[[VPC の前提知識]]"]
sources:
  - https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/hosted-zone-private-considerations.md
  - https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resolver.md
  - https://docs.aws.amazon.com/vpc/latest/userguide/vpc-dns.md
---

# DNS と名前解決

## ひとことで

VPC の中の名前解決は Route 53 Resolver が担い、外の DNS とつなぐときはエンドポイントの向きで使う道具が変わる。

## なぜ効くか / どこで使うか

ハイブリッド構成でもマルチアカウントでも、最初に詰まるのは名前解決だ。IP は通っているのに名前が引けないときは、この仕組みを確認すれば原因を切り分けられる。

オンプレの社内ドメインを AWS から引きたい、AWS の名前をオンプレから引きたい、別アカウントのゾーンを共有したい、という3つの要求で使う仕組みが違う。

## 前提となる知識

IP・CIDR と DNS の基礎は [[VPC の前提知識]] にある。

| 用語 | 一言の説明 |
|---|---|
| DNS の仕組み | 権威サーバと再帰サーバの役割分担。AWS の構成にもこの分担が対応する。 |
| 転送（フォワーディング） | 「このドメインはこの DNS に聞く」という振り分け。 |
| VPC の CIDR | Resolver のアドレスが「プライマリ CIDR の先頭 +2」に決まっている（[[VPC の前提知識]]）。 |
| パブリックとプライベートの違い | 名前が公開されるかどうかの違い。区別を誤ると社内の名前が公開される。 |

## 本体の解説

向きだけ先に押さえる。同じ通信でも、問い合わせがどちらへ流れるかで、使うエンドポイントが変わる。

![[assets/web/aws-dns-directions.svg]]

### 1. VPC の中の Resolver

VPC には Route 53 Resolver が用意されていて、プライマリ CIDR の先頭アドレスに +2 したアドレスにいる。決まりの理由と +1 の役割は [[VPC の前提知識]] にある。169.254.169.253 でも同じものに届く。

VPC の DNS 属性は2つあり、役割が違う。

| 属性 | 役割 | 無効のときの結果 |
|---|---|---|
| enableDnsSupport | VPC で Route 53 Resolver を使えるようにする | Resolver に問い合わせられず、名前解決そのものができない |
| enableDnsHostnames | インスタンスに DNS ホスト名を付ける（パブリック IP があればパブリック DNS 名も） | ホスト名が付かない |

プライベートホストゾーンを使うには、両方を true にする必要がある。CLI や CloudFormation で作った VPC では enableDnsHostnames が最初 false なので、ここでつまずきやすい。デフォルト VPC では最初から true になっている。

### 2. プライベートホストゾーン

Route 53 のプライベートホストゾーンは、VPC に関連付けることでその VPC から引けるようになる。関連付いていない VPC からは引けない。

同じアカウントの複数の VPC で使いたいだけなら、それぞれの VPC をゾーンに関連付ければよい。

別アカウントの VPC で使いたいときは2段階になる。まずゾーンを持つアカウントで、その VPC との関連付けを承認する。次に VPC を持つアカウントで関連付ける。承認がないと、VPC 側の関連付けはエラーになる。この操作はコンソールではできず、CLI や API で行う。

プライベートホストゾーンそのものは RAM で共有できない。多数のアカウントや VPC に配りたいときは、ゾーンや Resolver ルールを Route 53 Profiles にまとめ、その Profile を RAM で共有する。

### 3. オンプレとつなぐ向き

ここが一番混同しやすい。使うエンドポイントは、問い合わせの向きで決まる。

| やりたいこと | 使うもの |
|---|---|
| VPC からオンプレの社内ドメインを引きたい | アウトバウンドエンドポイント＋転送ルール |
| オンプレから AWS のプライベート名を引きたい | インバウンドエンドポイント |
| 両方向に引きたい | 両方を作る |

アウトバウンドは「VPC からの問い合わせを外の DNS へ送り出す出口」で、転送ルールにドメインと転送先 DNS を書く。インバウンドは「外からの問い合わせを VPC の Resolver で受ける入口」で、オンプレの DNS 側に、このエンドポイントの IP への条件付き転送を設定する。

DHCP オプションセットを書き換えて、VPC 内の DNS をオンプレ DNS に向ける方法もある。ただし VPC 内の問い合わせがすべてオンプレに行く。オンプレ DNS が AWS のインバウンドエンドポイントへ転送し直さない限り、プライベートホストゾーンや AWS の内部名は引けなくなる。通常は選ばない。

### 4. Resolver ルールの共有

転送ルールは VPC に関連付けて使う。多数の VPC で同じルールを使うなら、RAM で共有して各 VPC に関連付ける。

中央のネットワークアカウントにエンドポイントを置き、他のアカウントの VPC から共有する形が定番になる。

## 具体例・コード

オンプレの `corp.example.com` を VPC から引く場合を、4つの手順で組む。

1. アウトバウンドエンドポイントを、VPC の2つの AZ のサブネットに作る
2. 転送ルールを作り、`corp.example.com` の転送先にオンプレの DNS の IP を書く
3. ルールを対象の VPC に関連付ける
4. セキュリティグループと、オンプレ側のファイアウォールで DNS（UDP/TCP 53）を通す

## つまずきポイント

- 「インバウンドは VPC から外へ出す向き」は間違い。向きは問い合わせが流れる向きで決まる。オンプレから AWS の名前を引く問い合わせは VPC に入ってくるので、インバウンドになる。
- 「プライベートホストゾーンは関連付ければ引ける」は間違い。関連付けに加えて、DNS 属性が2つとも有効でないと引けない。
- 「社内の名前をパブリックホストゾーンに置いてよい」は間違い。インターネットに公開される。
- 「VPC CIDR の +1 を DNS に指定する」は間違い。+1 はルーター、DNS はプライマリ CIDR の先頭 +2。
- 「別アカウントのゾーンは VPC 側の操作だけで引ける」は間違い。ゾーン側の承認が先に要る。
- 「DHCP オプションセットでオンプレ DNS に向ければ足りる」は間違い。VPC 内の問い合わせがすべてオンプレに行き、オンプレ DNS が AWS へ転送し直さない限り、プライベートホストゾーンや AWS の内部名が引けなくなる。

## 疑問・TODO

- [ ] Route 53 Profiles で共有する場合と、関連付けを承認する場合の使い分けを整理する
- [ ] Resolver のクエリログで、どの問い合わせがどこへ流れたか追えるようにする
- [ ] オンプレ側の DNS が落ちたときの挙動（タイムアウトと再試行）を確認する

## 関連ノート

- [[VPC とネットワークの基礎]] — サブネットと経路の前提
- [[つなぎ方の選択]] — オンプレとつなぐ物理的な経路
- [[VPC の前提知識]] — CIDR と Resolver のアドレスの決まり方

## 出典

- https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/hosted-zone-private-considerations.md
- https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resolver.md
- https://docs.aws.amazon.com/vpc/latest/userguide/vpc-dns.md
