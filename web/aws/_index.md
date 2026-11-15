# AWS（SAP-C03 の勉強から出てきた仕組み）

試験対策の判断は別リポジトリ（AWS-SAP）に置き、ここには**仕組みの理解**だけを書く。

- [[VPC の前提知識]] — IP・CIDR・最長一致・プライベート IP・NAT・DNS の土台（先に読む）
- [[AWS の VPC とネットワークの基礎]] — サブネット、ルートテーブル、NAT、エンドポイント、SG と NACL
- [[AWS の DNS と名前解決]] — Route 53 Resolver、プライベートホストゾーン、転送の向き
- [[AWS のつなぎ方の選択]] — ピアリング、Transit Gateway、Direct Connect、VPN、PrivateLink
- [[AWS のロードバランサの使い分け]] — ALB、NLB、GWLB の層と用途
- [[AWS のマルチアカウントのガバナンス]] — Organizations、SCP、Control Tower、Identity Center
- [[AWS の IAM と権限の仕組み]] — ポリシーの評価順、ロールと AssumeRole、境界・SCP・セッション、クロスアカウント
