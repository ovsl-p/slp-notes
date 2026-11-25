---
title: IaC でインフラを揃える仕組み
aliases: [IaC でインフラを揃える仕組み, Infrastructure as Code, CloudFormation, CDK, StackSets, ドリフト検出, 変更セット, 動的参照, ポリシー・アズ・コード]
domain: web
tags: [aws, iac, cloudformation, cdk, stacks, governance, sap-c03]
created: 2026-11-25
updated: 2026-11-25
status: growing
prerequisites: [IAM の基本, マルチアカウントの考え方]
related: ["[[マルチアカウントのガバナンス]]", "[[組織の統制サービスの違い]]", "[[暗号化と秘密情報の管理]]", "[[検知と監査の違い]]"]
sources:
  - https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/Welcome.md
  - https://docs.aws.amazon.com/cdk/v2/guide/home.md
  - https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-updating-stacks-changesets.md
  - https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-stack-drift.md
  - https://docs.aws.amazon.com/cloudformation-cli/latest/hooks-userguide/what-is-cloudformation-hooks.html
  - https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/what-is-cfnstacksets.md
  - https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/dynamic-references.md
  - https://docs.aws.amazon.com/cdk/v2/guide/languages.md
---

# IaC でインフラを揃える仕組み

## ひとことで

IaC は、インフラの構成をコードで宣言し、同じ状態を何度でも再現する方法である。CloudFormation と CDK で作り、Hooks や SCP で止め、StackSets で配る。

## なぜ効くか / どこで使うか

テンプレートに構成を書くと、EC2 や RDS を個別に作って結線する手作業がなくなる。依存関係の解決も CloudFormation が担う。

止める仕組みをコードの段階に置くと、非準拠のリソースが本番に出る前に弾ける。秘密をテンプレートに書かない設計にすると、テンプレートを共有しても漏えいしない。

使う場面は次の3つである。

- 複数アカウントで同じ基盤を揃えるとき
- 暗号化などの社内ルールを全リソースに強制するとき
- パスワードを含む構成をリポジトリで管理するとき

アカウント統制の全体像は [[マルチアカウントのガバナンス]] と [[組織の統制サービスの違い]] にある。

## 前提となる知識

| 用語 | 一言の説明 |
|---|---|
| テンプレート | リソースとプロパティを記述したファイル。 |
| スタック | テンプレートから作られるリソースのまとまり。 |
| 変更セット | 更新が既存リソースに与える影響を、実行前に見せる機能。 |
| ドリフト検出 | CloudFormation の外で直接変えられた設定を見つける機能。 |
| 合成（synth） | CDK のコードからテンプレートを生成する工程。 |
| 動的参照 | デプロイ時に Secrets Manager や Parameter Store の値を取る書き方。 |

## 本体の解説

### 1. 宣言型と命令型（CloudFormation / CDK）

宣言型は、最終的にどうなっていてほしいかを書く方式である。差分の計算はサービス側が行う。命令型は、この順でこれをせよと手続きで書く方式である。

CloudFormation は宣言型である。CDK はプログラミング言語で書くが、生成物は CloudFormation テンプレートである。そのため、デプロイは宣言型の仕組みに乗る。

CDK は Construct Library（部品）、CLI、スタックとアプリの組み合わせで成り立つ。TypeScript、JavaScript、Python、Java、C#/.NET、Go の6言語で書ける。synth でテンプレートを生成し、CloudFormation 経由でデプロイする。

更新の前には変更セットを使う。リソースの置換（再作成）が起きるかを、実行前に確認できる。

実環境がテンプレートからずれた（ドリフトした）リソースがあると、その後の更新や削除が思わぬ結果になりやすい。ドリフト検出を定期的に実行して、ずれを見つけておく。

### 2. ポリシーをコードで強制する

統制の仕組みは4つあり、働く段階が違う。

| 仕組み | 働く段階 | 働き |
|---|---|---|
| CDK Aspects | 合成時 | スコープ内の全 Construct を巡回し、共通の検査や設定を適用する |
| CloudFormation Hooks | デプロイ時 | 作成・更新・削除の直前に検査し、FAIL モードなら止める（WARN は警告だけ） |
| AWS Config ルール | 運用中 | デプロイ済みのリソースがルールに合うかを継続的に評価する。プロアクティブモードならデプロイ前にも評価する（止めはしない） |
| SCP | API 呼び出し時 | メンバーアカウントのプリンシパルに操作の上限をかける（管理アカウントとサービスリンクロールには効かない。[[組織の統制サービスの違い]]） |

Aspects は、たとえば全バケットに暗号化を強制する用途に使う。Hooks はアノテーションで判定理由と修正案を返せる。

Config ルールは止めずに、デプロイ済みのリソースを評価して検知する。検知と監査の整理は [[検知と監査の違い]] を参照する。

### 3. シークレットをテンプレートに書かない

動的参照を使うと、テンプレートに秘密を書かずに済む。値はデプロイ時に取得される。

動的参照には `{{resolve:ssm:...}}`、`{{resolve:ssm-secure:...}}`、`{{resolve:secretsmanager:...}}` の3種類の書式がある。秘密に使うのは後の2つで、`ssm` は平文のパラメータを読む。パスワードはテンプレートにもパラメータにも平文で置かない。ローテーションは Secrets Manager に任せる。

動的参照は、リソースの主識別子（名前など）には使わない。解決後の平文の値が主識別子に使われ、スタックの出力などに現れることがあるからだ。

鍵と秘密情報の扱いは [[暗号化と秘密情報の管理]] にまとめてある。

### 4. StackSets で複数アカウント・複数リージョンへ配る

StackSets は、管理者アカウントから1つのテンプレートで、複数アカウント・複数リージョンにスタックを作成・更新・削除する。サービス管理モデルなら、Organizations の管理アカウントか、委任された管理者アカウントが管理者になる。

権限モデルは2つある。

| モデル | 内容 |
|---|---|
| サービス管理 | Organizations と連携する |
| 自己管理 | 自分で IAM ロールを用意する |

Organizations と連携するモデルなら、新規アカウントへの自動配備もできる。

## 具体例・コード

```yaml
Resources:
  Vpc:
    Type: AWS::EC2::VPC
    Properties: { CidrBlock: 10.0.0.0/16 }
  Bucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketEncryption:
        ServerSideEncryptionConfiguration:
          - ServerSideEncryptionByDefault: { SSEAlgorithm: aws:kms }
  Db:
    Type: AWS::RDS::DBInstance
    Properties:
      MasterUserPassword: '{{resolve:secretsmanager:MySecret:SecretString:password}}'
```

CDK では、同じ内容を Construct の組み合わせで書く。バケット部分は、synth すると上のテンプレートに近い YAML が出る。`KMS` を選ぶと暗号化キーを新しく作るため、`AWS::KMS::Key` が1つ増える。

```ts
const bucket = new s3.Bucket(this, 'Data', {
  encryption: s3.BucketEncryption.KMS_MANAGED,
});
```

バケットの暗号化を全スタックで強制したいなら、Aspects で Bucket を巡回して設定を足す。テンプレートを1つずつ直す必要はない。

## つまずきポイント

- 「CDK は命令型だからデプロイも手続き的に動く」は間違い。生成物が CloudFormation テンプレートなので、デプロイは宣言型の仕組みに乗る。
- 「変更セットを作った時点で、変更は適用される」は間違い。変更セットは影響を見せるだけで、実行を決めるまで適用されない。
- 「Config ルールで非準拠のデプロイを止められる」は間違い。Config ルールはデプロイ済みのリソースを継続的に評価する。止めるのは合成時なら Aspects、デプロイ時なら Hooks である。
- 「SCP はテンプレートの中身を検査する」は間違い。SCP はメンバーアカウントのプリンシパルに操作の上限をかける。管理アカウントとサービスリンクロールには効かない。
- 「動的参照はリソース名にも使ってよい」は間違い。主識別子に使うと、平文の値がスタックの出力などに現れることがある。
- 「StackSets は自己管理の権限モデルでも、新規アカウントに自動で配備できる」は間違い。自動配備は、Organizations と連携するサービス管理の権限モデルでだけ使える。

## 疑問・TODO

- [ ] Hooks と Config ルールを同じ要件に両方入れる場合の使い分けを整理する
- [ ] StackSets で一部のアカウントだけ失敗したときの扱いを確認する

## 関連ノート

- [[マルチアカウントのガバナンス]] — アカウント分割と、統制を置く層の全体像
- [[組織の統制サービスの違い]] — SCP と Organizations の機能の違い
- [[暗号化と秘密情報の管理]] — Secrets Manager と Parameter Store の使い分け
- [[検知と監査の違い]] — Config と CloudTrail の役割の違い

## 出典

- https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/Welcome.md
- https://docs.aws.amazon.com/cdk/v2/guide/home.md
- https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-updating-stacks-changesets.md
- https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-stack-drift.md
- https://docs.aws.amazon.com/cloudformation-cli/latest/hooks-userguide/what-is-cloudformation-hooks.html
- https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/what-is-cfnstacksets.md
- https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/dynamic-references.md
- https://docs.aws.amazon.com/cdk/v2/guide/languages.md
