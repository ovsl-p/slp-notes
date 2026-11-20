---
title: 責任ある AI の統制
aliases: [AWS の責任ある AI の統制, 責任あるAI, responsible AI, AI ガバナンス, Guardrails, AgentCore Identity, ヒューマンインザループ, 人の承認, Bedrock ガードレール]
domain: web
tags: [aws, ai, generative-ai, guardrails, agentcore, identity, governance, human-in-the-loop, sap-c03]
created: 2026-11-20
updated: 2026-11-20
status: growing
prerequisites: [基盤モデルとトークン, ベクトル検索]
related: ["[[AI サービスの使い分け]]", "[[IAM と権限の仕組み]]", "[[検知と監査の違い]]", "[[組織の統制サービスの違い]]"]
sources:
  - https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails.html
  - https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-tiers.html
  - https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-contextual-grounding-check.html
  - https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-automated-reasoning-checks.html
  - https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-use-independent-api.html
  - https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/identity.html
  - https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/identity-overview.html
  - https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/inbound-jwt-authorizer.html
  - https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/identity-outbound-credential-provider.html
  - https://docs.aws.amazon.com/step-functions/latest/dg/connect-to-resource.html
  - https://docs.aws.amazon.com/step-functions/latest/dg/tutorial-human-approval.html
  - https://docs.aws.amazon.com/wellarchitected/latest/agentic-ai-lens/agentsec04-bp02.html
  - https://docs.aws.amazon.com/bedrock/latest/userguide/model-invocation-logging.html
---

# 責任ある AI の統制

## ひとことで

責任ある AI の統制は、入出力の検査（Guardrails）、実行権限（AgentCore Identity）、人の承認（Step Functions）の3層で作る。

## なぜ効くか / どこで使うか

責任ある AI は、英語の responsible AI の直訳で、技術名ではなく運用の方針を指す。分野2のタスク2.5は、その方針を実際の部品に落とす設問である。設問は「どこで止めるか」を問うてくる。出力の内容か、実行できる操作か、それとも決定そのものかで、使うサービスが変わる。

## 前提となる知識

| 用語 | 一言の説明 |
|---|---|
| ガードレール | 入力と出力を検査し、危険な内容や個人情報を止める仕組み。 |
| プロンプトインジェクション | 入力文に指示を紛れ込ませ、モデルの動作を変える攻撃。 |
| ワークロードアイデンティティ | 人ではなく、アプリやエージェントに与える ID。 |
| JWT | 署名付きのトークン。発行元と権限を検証できる。 |
| スコープ | トークンに含める権限の範囲。 |
| オンビハーフオブ | 利用者の権限を借りて、代理で操作すること。 |
| コールバック | 処理をいったん止め、外部からの応答で再開する方式。 |
| 承認者 | 人間の確認を担当する人。利用者本人とは限らない。 |

## 本体の解説

### 1. 3つの層で止める

| 層 | 問い | 主なサービス |
|---|---|---|
| 入出力の内容 | 危険な内容や個人情報が入っていないか | Bedrock Guardrails |
| 実行できる操作 | 誰の権限で、どこまで操作させるか | AgentCore Identity |
| 決定そのもの | 人の確認を挟むべき操作か | Step Functions、AgentCore Runtime |

下の層ほど止める力が強い。内容の検査は言い換えで抜けられるが、権限を絞れば操作自体が通らない。

### 2. Guardrails で入出力を検査する

Guardrails は、モデルに渡す入力と、モデルが返す出力の両方を検査する。検査の種類は6つある。

| 検査 | 止めるもの |
|---|---|
| コンテンツフィルタ | ヘイト、侮辱、性的、暴力、不正行為、プロンプト攻撃の6分類 |
| 拒否トピック | 事業として扱わない話題 |
| 単語フィルタ | 競合名や禁止語。大文字と小文字を区別せず、語単位で一致させる |
| 機密情報フィルタ | 個人情報の検出とマスク。正規表現も足せる |
| 文脈グラウンディング検査 | 参照文書に無い内容や、質問と無関係な答え |
| 自動推論チェック | 決めた規則に反する記述 |

コンテンツフィルタは文章だけでなく画像にも効く。フィルタの強度は分類ごとに変えられる。

文脈グラウンディング検査は、RAG の出力が参照文書に根拠を持っているかを見る。自動推論チェックは、規則を論理式に落として、応答が規則に反するかどうかを判定する。「たぶん正しい」ではなく、規則に照らして正しいかを示せる点が違う。

Guardrails には Classic と Standard の2つの水準がある。Standard では、コードのコメントや変数名に紛れ込ませた有害な内容も検出し、対応する言語も増える。

使い方は2通りある。モデル呼び出しの引数 `guardrailIdentifier` に指定する方法と、ApplyGuardrail API を自分で呼ぶ方法だ。後者なら、Bedrock 以外のモデルや、モデルを通さない入力にも同じ検査をかけられる。

### 3. AgentCore Identity で実行権限を絞る

エージェントは人ではないので、IAM ユーザーを配るわけにいかない。AgentCore Identity は、エージェント用のワークロードアイデンティティを発行し、認証と権限をまとめて扱う。

入口は inbound の JWT オーソライザーが担う。Runtime と Gateway は、設定した discovery URL から公開鍵を取り、届いた JWT を検証する。IdP を選ばないので、Cognito、Okta、Microsoft Entra ID、Auth0 のどれでも使える。許可するスコープやクレームは設定で絞る。

出口はクレデンシャルプロバイダーが担う。OAuth 2.0 のアクセストークン、API キー、クライアント証明書、SAML アサーションを預かり、必要になったときに取り出す。エージェントには長期の秘密を渡さず、取り出しの記録を残す。

利用者の代理で操作する場合は、同意を取る画面（同意ポータル）を挟める。誰の権限で何をしたかが追えるので、監査の証跡として残る。

### 4. Step Functions で人の承認を挟む

金額の確定や削除のように、取り返しのつかない操作は、モデルの判断だけで実行させない。Step Functions のコールバック方式（`waitForTaskToken`）を使うと、処理をいったん止めて、承認者の応答が返るまで待てる。

流れは3段で進む。Step Functions がタスクトークンを発行して外部に渡す。外部のアプリが承認者に確認を出す。承認者が答えると、アプリがトークン付きで応答を返し、処理が再開する。承認の画面はメールでも申請システムでもよく、Step Functions は待つだけになる。

承認の置き場所は、エージェントの作りで変わる。Agents Classic には、関数を実行する前に確認を求める仕組みと、関数呼び出し自体をアプリに返す仕組み（return of control）がある。どちらも利用者本人が承認者になる対話向けだ。別の担当者が承認する非同期の業務なら、Step Functions のコールバックが向く。何時間もかかる承認を待たせたい場合は、AgentCore Runtime の非同期実行に載せる。

### 5. 記録を残す

モデルの呼び出しは、モデル呼び出しログを S3 か CloudWatch Logs に出す。既定では無効で、保存先は同じアカウント、同じリージョンに限られる。

エージェントの実行の流れは、AgentCore Observability が OpenTelemetry 形式で出す。どの道具を、どの順に、どの入力で呼んだかが残る。IAM や Organizations の操作は、これまでどおり CloudTrail と Config で追う。

### 6. 誰の権限で動かすかを決める

エージェントが操作するとき、権限の出どころは3つある。エージェント自身のロール、利用者の権限を借りる形、あいだのサービスロールの3つだ。利用者の権限を借りるなら、その利用者が持たない操作はエージェントにもできない。

複数の利用者を1つのエージェントで受けるときは、セッションを分ける。AgentCore Runtime はセッションを分離するので、会話の記憶が別の利用者に混ざらない。

## 具体例・コード

返金を受け付けるエージェントを例にする。

1. 利用者の JWT を inbound オーソライザーが検証し、注文の参照権限だけを持つトークンだと確かめる。
2. 入力の個人情報を、Guardrails の機密情報フィルタでマスクする。
3. 返金の可否は、社内の規程を入れたナレッジベースで調べる。出力は文脈グラウンディング検査に通す。
4. 返金 API の呼び出しは、AgentCore Identity が預かる短期のトークンで行う。
5. 金額が閾値を超えたら、Step Functions のコールバックで承認者に回す。承認が返るまで返金は実行しない。
6. 呼び出しの記録はモデル呼び出しログと Observability に残す。

## つまずきポイント

- 「Guardrails を有効にすれば、すべての呼び出しに自動で効く」は間違い。リクエストで `guardrailIdentifier` を指定するか、ApplyGuardrail を自分で呼ぶ。
- 「プロンプトの検査だけで十分」は間違い。言い換えで検査を抜けられるので、実行できる操作を Identity で絞る層が要る。
- 「承認は利用者本人がやるもの」は間違い。非同期の業務では別の担当者が承認する。Step Functions のコールバックは、その形に向く。
- 「エージェントには IAM ユーザーを作って鍵を渡す」は間違い。ワークロードアイデンティティと短期のクレデンシャルを使い、長期の鍵を持たせない。
- 「モデル呼び出しログは既定で残る」は間違い。既定は無効で、自分で有効にする。

## 疑問・TODO

- [ ] ガードレールを複数アカウントで共有できるかを確認する
- [ ] 自動推論チェックの料金と、扱える規則の書き方を確認する
- [ ] Guardrails の Standard と Classic の違いを、対応モデルとリージョンまで整理する

## 関連ノート

- [[AI サービスの使い分け]] — Bedrock、ナレッジベース、AgentCore の役割
- [[IAM と権限の仕組み]] — 人の権限をどう絞るか。エージェントの権限はこの考え方の延長にある
- [[検知と監査の違い]] — 呼び出しの記録をどこに集め、どう評価するか
- [[組織の統制サービスの違い]] — 組織全体で禁止事項を効かせる仕組み

## 出典

- https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails.html
- https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-tiers.html
- https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-contextual-grounding-check.html
- https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-automated-reasoning-checks.html
- https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-use-independent-api.html
- https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/identity.html
- https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/identity-overview.html
- https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/inbound-jwt-authorizer.html
- https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/identity-outbound-credential-provider.html
- https://docs.aws.amazon.com/step-functions/latest/dg/connect-to-resource.html
- https://docs.aws.amazon.com/step-functions/latest/dg/tutorial-human-approval.html
- https://docs.aws.amazon.com/wellarchitected/latest/agentic-ai-lens/agentsec04-bp02.html
- https://docs.aws.amazon.com/bedrock/latest/userguide/model-invocation-logging.html
