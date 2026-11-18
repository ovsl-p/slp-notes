---
title: AWS の監査証拠の提出
aliases: [AWS の監査証拠の提出, AWS の証跡の提出, 証跡の提出, 監査の証拠, Audit Manager, Artifact, AWS Artifact, エビデンス, コンプライアンス資料, 合意書]
domain: web
tags: [aws, compliance, audit, audit-manager, artifact, sap-c03]
created: 2026-11-18
updated: 2026-11-18
status: growing
prerequisites: ["[[AWS の IAM と権限の仕組み]]"]
related: ["[[AWS の検知と監査の違い]]", "[[AWS の組織の統制サービスの違い]]", "[[AWS のマルチアカウントのガバナンス]]"]
sources:
  - https://docs.aws.amazon.com/audit-manager/latest/userguide/what-is-audit-manager.html
  - https://docs.aws.amazon.com/audit-manager/latest/userguide/how-evidence-is-collected.html
  - https://docs.aws.amazon.com/audit-manager/latest/userguide/assessment-reports.html
  - https://docs.aws.amazon.com/audit-manager/latest/userguide/audit-manager-availability-change.html
  - https://aws.amazon.com/artifact/faq
  - https://docs.aws.amazon.com/artifact/latest/ug/manage-org-agreement.html
  - https://docs.aws.amazon.com/artifact/latest/ug/what-is-aws-artifact.html
---

# AWS の監査証拠の提出

## ひとことで

Audit Manager は自分の AWS の使い方の証拠を集める。Artifact は AWS 基盤の証拠を取り出す。

## なぜ効くか / どこで使うか

監査では、規格が求める証拠を毎回手作業で集めると時間がかかる。両者は証拠の出どころが違うため、提出先に応じて使い分ける必要がある。

このメモでは、監査に出す証拠の二つの出どころと使い分けを扱う。AWS 基盤の証拠と自分の利用の証拠は別物だと分かることが狙いだ。

## 前提となる知識

| 用語 | 一言の説明 |
|---|---|
| 監査 | 規格どおりに運用されているかを確かめる作業である。 |
| 規格（フレームワーク） | 監査の基準となる枠組みである。 |
| 管理策（コントロール） | 規格が求める個々の対策項目である。 |
| 証拠（エビデンス） | 管理策を満たしていることを示す記録である。 |
| 評価（アセスメント） | Audit Manager で証拠を集める対象の単位である。 |
| コンプライアンス資料 | 規格への適合を示す文書（レポート）である。 |
| 合意書（BAA など） | 顧客と AWS が取り交わす契約文書である。 |
| 管理アカウント | 組織配下のアカウントを束ねる親のアカウントである。 |
| スナップショット | ある時点の設定状態を写し取ったものである。 |

## 本体の解説

### Audit Manager の役割

Audit Manager は、評価の対象アカウントを決めると証拠の収集を始める。
自動で集めるのは三種類で、それぞれ出どころが決まっている。コンプライアンスチェックは Security Hub CSPM と Config の評価結果、ユーザー操作は CloudTrail のログ、設定データは各サービスの describe 系 API の結果から取る。
これとは別に、自分でアップロードした証拠（手動証拠）も管理策に紐付けられる。
集めた情報は証拠に変換され、管理策に紐付けられる。
レポートには、集めた証拠のうち自分で選んだものだけが入る。

### AWS Artifact の役割

Artifact は、AWS 側のコンプライアンス資料と合意書を取り出すポータルである。
取り出せるのは主に、AWS 基盤が規格を満たしていることの証拠である。Marketplace の ISV が出したレポートも、Vendor Insights でアクセスを許可されていれば見られる。
顧客自身の利用の証拠ではない。
組織の管理アカウントで合意書を受け入れると、既存と今後のメンバーアカウントすべてに自動で適用される。組織が「すべての機能」で有効になっていることが前提である。

### 両者の違い

Artifact は「AWS 基盤が規格を満たしている」ことを示す。
Audit Manager は「自分の AWS の使い方が規格を満たしている」ことを示す。
監査では両方を組み合わせて提出する場面がある。

### 注意点

Audit Manager は 2026年4月30日から、新しいアカウントでセットアップできなくなった。メンテナンスモードに入り、新機能や新しいフレームワークの追加は行われない。
単一アカウントで使っていた既存の利用者は、そのアカウントとリージョンの中でなら新しい評価も作れるが、リージョンやアカウントを増やせない。組織で使っていた利用者は、組織内のアカウントを評価に加えられるが、リージョンは増やせない。
代替として、AWS は Config の適合パック、Control Tower、Security Hub CSPM を挙げている。Config の適合パックは Config ルールの評価だけを証拠にするので、CloudTrail のログや Security Hub CSPM の制御は集めない。

## 具体例

1. 監査で求められている規格と管理策を確認する。
2. 各管理策が AWS 基盤の話か、自分の使い方の話かを切り分ける。
3. 基盤の話なら、Artifact で該当のレポートや合意書を取り出す。
4. 使い方の話なら、Audit Manager で評価の対象アカウントを決める。既存の利用者の場合の手順で、新しく始めるなら Config の適合パックなどで代える。
5. 証拠が管理策に紐付いているかを確認する。
6. レポートに入れる証拠を選び、提出前に内容を確認する。

## つまずきポイント

- Artifact のレポートがあれば、自分の証拠も足りると思う。実際は、基盤の証拠にすぎず、利用の証拠は別に必要になる。
- Audit Manager を入れれば規格に準拠すると思う。実際は、証拠を集めるだけであり、準拠しているかの判断は別途必要になる。
- 組織の管理アカウントで合意すれば、配下の証拠がすべて揃うと思う。実際は、合意の適用は配下に及ぶが、自分の利用の証拠は含まれない。
- Audit Manager は今から誰でも使えると思う。実際は、2026年4月30日から新しいアカウントではセットアップできない。
- 設定のスナップショットがあれば、ユーザー操作の履歴も揃うと思う。実際は、設定の状態とユーザー操作は別の種類の証拠である。

## 疑問・TODO

- [x] Audit Manager の新規受付状況が最新でどうなっているかを確認する
- [x] 新規顧客が同等の証拠収集を行う代替手段を確認する
- [ ] Artifact で取り出せる資料の種類が、対象の規格に含まれるかを確認する

## 関連ノート

- [[AWS の検知と監査の違い]] — 検知と監査で求められる証拠の性質の違い
- [[AWS の組織の統制サービスの違い]] — 組織単位で統制を効かせる仕組みとの対比
- [[AWS のマルチアカウントのガバナンス]] — 管理アカウントと配下アカウントの考え方

## 出典

- https://docs.aws.amazon.com/audit-manager/latest/userguide/what-is-audit-manager.html
- https://docs.aws.amazon.com/audit-manager/latest/userguide/how-evidence-is-collected.html
- https://docs.aws.amazon.com/audit-manager/latest/userguide/assessment-reports.html
- https://docs.aws.amazon.com/audit-manager/latest/userguide/audit-manager-availability-change.html
- https://aws.amazon.com/artifact/faq
- https://docs.aws.amazon.com/artifact/latest/ug/manage-org-agreement.html
- https://docs.aws.amazon.com/artifact/latest/ug/what-is-aws-artifact.html
