---
title: AWS のネットワークサービスの違い
aliases: [AWS のネットワークサービスの違い, CloudFront, Global Accelerator, Route 53, routing policy, Transit Gateway, Direct Connect, Site-to-Site VPN, PrivateLink]
domain: web
tags: [aws, network, cloudfront, global-accelerator, route53, transit-gateway, direct-connect, sap-c03]
created: 2026-11-16
updated: 2026-11-20
status: growing
prerequisites: [IP アドレスと CIDR, DNS の仕組み, ロードバランサの層（L4 と L7）]
related: ["[[AWS の VPC とネットワークの基礎]]", "[[AWS の DNS と名前解決]]", "[[AWS のつなぎ方の選択]]", "[[AWS のロードバランサの使い分け]]", "[[VPC の前提知識]]"]
sources:
  - Amazon CloudFront デベロッパーガイド
  - AWS Global Accelerator デベロッパーガイド
  - Amazon Route 53 デベロッパーガイド（ルーティングポリシー、エイリアスレコード）
  - AWS Transit Gateway ユーザーガイド
  - AWS Direct Connect / Site-to-Site VPN ユーザーガイド
---

# AWS のネットワークサービスの違い

## ひとことで

「誰が近くから読むか」「どの層で振り分けるか」「どこまで経路を流すか」の3つで、ネットワークのサービスは分かれる。

## なぜ効くか / どこで使うか

分野1の設問は、通信をどこで受け、どこへ流し、どう切り替えるかを選ばせる。サービス名だけ覚えても、「速くしたい」「止まりたくない」「社内だけにしたい」といった要件に当てはめられない。

## 前提となる知識

[[VPC の前提知識]] の IP・CIDR・ルーティングが分かっていればよい。ロードバランサの L4（TCP）と L7（HTTP）の違いは [[AWS のロードバランサの使い分け]] を参照。

## 本体の解説

### 1. 3つの軸で分ける

| 軸 | 問い | 代表的なサービス |
| --- | --- | --- |
| 近くで受けて配る | 利用者に近い場所でキャッシュや終端をするか | CloudFront、Global Accelerator |
| 名前から振り分ける | どの IP に案内するか（DNS） | Route 53 |
| 中で流す | VPC や拠点の間をどう経路でつなぐか | Transit Gateway、Direct Connect、VPN、PrivateLink |

### 2. CloudFront と Global Accelerator

どちらも利用者に近い場所（エッジ）から AWS に入る。違いは中身を見るかどうか。

CloudFront は HTTP と HTTPS を扱い、コンテンツをエッジに保存（キャッシュ）する。静的な画像や動画に強い。S3 を配信元にするときは、OAC（Origin Access Control）を使う。これで S3 を直接公開せず、CloudFront 経由だけに絞れる。

Global Accelerator は固定の IP を2つ（anycast）持つ。TCP と UDP を、AWS のネットワーク内で目的地まで運ぶ。キャッシュはしない。ゲームや IoT の UDP、固定 IP を顧客に教える必要がある場合に向く。リージョンをまたぐ障害切り替えを速くしたいときも候補になる。

### 3. Route 53 のルーティングポリシー

DNS の答えを、条件によって変える仕組み。設問では条件語で選ぶ。

| ポリシー | 向く場面 |
| --- | --- |
| シンプル | 1つの宛先だけ |
| 加重 | 割合で分ける（カナリアリリースや移行の段階的切り替え） |
| レイテンシー | 利用者から一番速いリージョンへ |
| フェイルオーバー | 障害時にだけ待機系へ（ヘルスチェックと組み合わせる） |
| 地理的な場所 | 国や地域で分ける（法令やライセンス） |

エイリアスレコードは、ELB や CloudFront などの AWS リソースを名前で指す、Route 53 独自のレコード。

頂点（example.com そのもの）には CNAME を置けない。CNAME は「この名前は別の名前の別名だ」という記録で、その名前には他の記録を置けないというルールがある。一方、頂点には SOA と NS の記録が必ずあるので、CNAME を置くとこのルールと衝突する。

エイリアスレコードは CNAME と違い、Route 53 が問い合わせに応じて宛先の IP を返す仕組みなので、頂点にも置ける。`example.com` を CloudFront に向けるならエイリアス、`www.example.com` なら CNAME でも足りる。

### 4. 中で流す手段

Transit Gateway は拠点を集める中継のハブ。VPC や VPN、Direct Connect をアタッチして、ルートテーブルで流す先を分ける。ハブ経由で届くので、推移的（中継する）。VPC が増えるほど効く。

Direct Connect は専用線で、回線の帯域と遅延を安定させる。ただし暗号化は標準では無いので、暗号化が要るなら VPN を重ねるか、別の仕組みで暗号化する。冗長化は回線もロケーションも二重にする。

Site-to-Site VPN は、インターネットの上に暗号化されたトンネルを作る。速く安く作れるが、帯域と遅延は回線に依存する。トンネルは2本張って冗長化するのが基本。

PrivateLink は経路を流さず、特定のサービスだけを公開する。相手の VPC と CIDR が重なっていても使える。

### 5. 選び方の順番

1. 利用者に近い場所で受けたい → CloudFront（HTTP）か Global Accelerator（TCP・UDP、固定 IP）
2. どの宛先へ案内するか → Route 53 のルーティングポリシー
3. 拠点や VPC をつなぐ → 数が多ければ Transit Gateway、少なければピアリング
4. 帯域と遅延の安定が要る → Direct Connect、急ぎで安く → VPN
5. 特定のサービスだけ見せる → PrivateLink

### 6. データ転送のコスト

AWS のデータ転送は、受信は無料で送信は課金されるのが基本の原則である。設計では、データがどの経路で外に出るかを先に確認する。

| サービス | 課金される項目 | 補足 |
|---|---|---|
| CloudFront | 利用者へ配信するデータ転送料金 | オリジン（配信元のサーバー）からの取得は別に課金される |
| ゲートウェイエンドポイント | 追加料金なし | S3 と DynamoDB 向けの、VPC 内から使う閉じた経路 |
| インターフェースエンドポイント | AZ ごとの時間課金と、流した GB 課金 | PrivateLink を使う接続点 |
| NAT ゲートウェイ | 設置時間の課金と、通したデータ量の課金 | プライベートサブネットから外部へ出る変換装置 |
| Transit Gateway | アタッチ時間の課金と、流したデータ量の課金 | VPC などを集約する中継点 |

CloudFront では、利用者までの配信を CloudFront のデータ転送料金で払う。オリジンからの取得は別に課金される。Origin Shield（オリジン手前のキャッシュ層）を挟むと、オリジンへの取得をまとめられる。ただし Shield 自体の料金が加わる。

AZ（アベイラビリティゾーン）をまたぐ転送は課金される。同じ AZ 内なら課金されない。AZ を分ける設計は、可用性と引き換えに転送費が増える。

S3 のクロスリージョンレプリケーションは、リージョン間の転送とレプリケーションのリクエストが課金される。Bedrock のクロスリージョン推論は、別リージョンで処理される。そのため、データの所在地と転送の扱いが変わる。

## 具体例・コード

グローバル向けの Web サービスの例。

1. 静的な画像は S3 に置き、CloudFront（OAC）で配信する
2. 動的な API は ALB の背後に置き、CloudFront の別のオリジンにする
3. 独自ドメインは Route 53 のエイリアスレコードで CloudFront に向ける
4. 障害時に別リージョンへ切り替えたいなら、CloudFront のオリジングループ（オリジンフェイルオーバー）で別リージョンのオリジンへ切り替える。CloudFront を通さないなら、Route 53 のフェイルオーバーで、オリジンのホスト名のレコードを切り替える

## つまずきポイント

- Global Accelerator もキャッシュすると思う。キャッシュはしない。
- Direct Connect を使えば暗号化もされると思う。標準では暗号化されない。
- ゾーンの頂点に CNAME を置けると思う。置けないので、エイリアスレコードを使う。
- Route 53 の加重を、障害切り替えの仕組みだと思う。障害時の切り替えはフェイルオーバーで、加重は割合の制御。
- PrivateLink をピアリングの代わりに、いつでも使えると思う。経路を流すものではないので、通信の向きと用途が違う。

- 「受信は無料だから転送費は考えなくてよい」は間違い。送信側のデータ量が課金対象になるため、外向きの経路を先に確認する必要がある。
- 「ゲートウェイエンドポイントにも時間課金がかかる」は間違い。S3 と DynamoDB 向けのゲートウェイエンドポイントは追加料金がない。インターフェースエンドポイントとは課金の仕組みが別である。
- 「AZ を分けても転送費は変わらない」は間違い。AZ をまたぐ転送は課金されるため、可用性を上げるほど転送費が増える。

## 疑問・TODO

- [ ] CloudFront のキャッシュキーと、Global Accelerator のエンドポイント重みの違いを整理する
- [ ] Route 53 のヘルスチェックの判定条件（間隔と閾値）を確かめる
- [ ] Transit Gateway のルートテーブルを、共有サービス型で1つ書いてみる

## 関連ノート

- [[AWS の VPC とネットワークの基礎]] — 経路とサブネットの前提
- [[AWS の DNS と名前解決]] — Route 53 Resolver と名前解決の向き
- [[AWS のつなぎ方の選択]] — 拠点やサービスをどうつなぐか
- [[AWS のロードバランサの使い分け]] — ALB・NLB・GWLB の層

- [[AWS のコストを下げる購入とサイジング]] — 転送費は購入オプションとは別の軸で効く

## 出典

- Amazon CloudFront デベロッパーガイド
- AWS Global Accelerator デベロッパーガイド
- Amazon Route 53 デベロッパーガイド
- AWS Transit Gateway ユーザーガイド
- AWS Direct Connect / Site-to-Site VPN ユーザーガイド
- https://aws.amazon.com/vpc/pricing/
- https://aws.amazon.com/transit-gateway/pricing/
- https://aws.amazon.com/privatelink/pricing/
- https://docs.aws.amazon.com/vpc/latest/privatelink/gateway-endpoints.md
- https://aws.amazon.com/cloudfront/pricing/
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/replication.html
- https://docs.aws.amazon.com/bedrock/latest/userguide/cross-region-inference.md
