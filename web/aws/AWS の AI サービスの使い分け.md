---
title: AWS の AI サービスの使い分け
aliases: [AWS の AI サービスの使い分け, AWS の AI と ML の違い, Bedrock, Amazon Bedrock, SageMaker AI, ナレッジベース, RAG, AgentCore, 生成AI, 基盤モデル, 組み込みAI]
domain: web
tags: [aws, ai, generative-ai, bedrock, sagemaker, rag, agents, agentcore, guardrails, sap-c03]
created: 2026-11-20
updated: 2026-11-20
status: growing
prerequisites: [基盤モデルとトークン, ベクトル検索]
related: ["[[AWS の疎結合の部品の違い]]", "[[AWS のコンピューティングサービスの違い]]", "[[AWS の IAM と権限の仕組み]]", "[[AWS の暗号化と秘密情報の管理]]"]
sources:
  - https://docs.aws.amazon.com/bedrock/latest/userguide/what-is-bedrock.html
  - https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base.html
  - https://docs.aws.amazon.com/bedrock/latest/userguide/kb-build-managed.html
  - https://docs.aws.amazon.com/bedrock/latest/userguide/kb-managed-sync.html
  - https://docs.aws.amazon.com/bedrock/latest/userguide/kb-managed-cross-account.html
  - https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/what-is-bedrock-agentcore.html
  - https://docs.aws.amazon.com/bedrock/latest/userguide/agents.html
  - https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails.html
  - https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-contextual-grounding-check.html
  - https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-automated-reasoning-checks.html
  - https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-use-independent-api.html
  - https://docs.aws.amazon.com/bedrock/latest/userguide/model-access.html
  - https://docs.aws.amazon.com/bedrock/latest/userguide/cross-region-inference.html
  - https://docs.aws.amazon.com/bedrock/latest/userguide/batch-inference.html
  - https://docs.aws.amazon.com/bedrock/latest/userguide/prov-throughput.html
  - https://docs.aws.amazon.com/bedrock/latest/userguide/model-invocation-logging.html
  - https://docs.aws.amazon.com/bedrock/latest/userguide/vpc-interface-endpoints.html
  - https://docs.aws.amazon.com/bedrock/latest/userguide/model-customization.html
  - https://docs.aws.amazon.com/bedrock/latest/userguide/flows.html
  - https://docs.aws.amazon.com/sagemaker/latest/dg/deploy-model-options.html
  - https://docs.aws.amazon.com/sagemaker/latest/dg/whatis.html
  - https://aws.amazon.com/bedrock/pricing/
  - https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base-setup.md
  - https://docs.aws.amazon.com/prescriptive-guidance/latest/choosing-an-aws-vector-database-for-rag-use-cases/vector-db-options.html
---

# AWS の AI サービスの使い分け

## ひとことで

モデルを借りるなら Bedrock、自前のモデルを動かすなら SageMaker AI、自社データを根拠に足すならナレッジベース、定型の抽出は既製の AI サービスで受ける。

## なぜ効くか / どこで使うか

分野1のタスク1.2は、AI をアーキテクチャに組み込む設問である。設問は、モデルを学習から持ち込みたいのか、既製のモデルを使いたいのかを言ってくる。この違いを先に見分けると、答えるサービスが決まる。

設問が「自社データを根拠に答えさせたい」なら、モデルの選択より先に検索の設計が問われる。逆に「申請を自動で進めたい」なら、主題は道具のつなぎ方に移る。

## 前提となる知識

| 用語 | 一言の説明 |
|---|---|
| 基盤モデル | 大量のデータで事前学習した汎用のモデル。文章や画像の生成に使う。 |
| 推論 | 学習済みのモデルに入力を与えて、出力を得ること。 |
| トークン | モデルが入出力を数える単位。課金の単位でもある。 |
| 埋め込み | 文章を数値の並び（ベクトル）に変えること。意味の近さを距離で測れる。 |
| ベクトルストア | 埋め込みを保存し、近いものを検索できるデータベース。 |
| RAG | 検索で得た情報をプロンプトに足してから生成する手法。 |
| エージェント | モデルが道具（API）を選んで実行し、作業を進める仕組み。 |
| ツール呼び出し | モデルが外部の関数や API を呼ぶ出力を返す機能。 |
| ナレッジベース | データソース、埋め込み、ベクトルストア、検索をまとめた単位。 |
| ガードレール | 入力と出力を検査し、危険な内容や個人情報を止める仕組み。 |

## 本体の解説

### 1. 6つの選択肢

| 選ぶもの | サービス | 向く場面 |
|---|---|---|
| モデルを借りる | Amazon Bedrock | 既製の基盤モデルを API で使いたい |
| 自前のモデルを動かす | SageMaker AI のエンドポイント | 学習済みモデルや独自コンテナを持ち込みたい |
| 定型の処理を任せる | 組み込み AI（Comprehend など） | 分類、抽出、文字起こしなど出力の形が決まっている |
| 自社データを根拠にする | Bedrock ナレッジベース | 社内文書を根拠に答えさせたい |
| 道具を使わせる | Bedrock AgentCore | 申請や検索の操作をモデルに実行させたい |
| 入出力を検査する | Bedrock Guardrails | 不適切な内容や個人情報を止めたい |

### 2. Bedrock でモデルを借りる

Bedrock は、複数の提供元の基盤モデルを API 経由で使うフルマネージドのサービスである。100以上のモデルから選べ、GPU やサーバーを持たなくてよい。呼び出しには `bedrock-runtime` の Converse と InvokeModel を使う。Converse はモデルごとの差分を吸収する共通形式で、InvokeModel はモデル固有の形式を受け付ける。

課金はトークン単位で決まる。大量の入力をまとめて処理するバッチ推論は、対応するモデルならオンデマンドの50%の価格で使える。処理能力を固定したいときは、Provisioned Throughput をモデルユニット単位で購入する。コミットなし、1か月、6か月から選び、期間が長いほど時間単価が下がる。

サーバーレスのモデルは、対応している商用リージョンなら既定で使える。リージョンごとに有効化する手順はない。初回の呼び出しで AWS Marketplace の購読が自動で始まるので、呼ぶ IAM 主体に `aws-marketplace:Subscribe` などの権限が要る。許可はアカウントごとに別で、Anthropic の Claude Opus 4.7 より前のモデルは初回利用フォームの提出を求められる。地理単位かグローバルの推論プロファイルを使うと、複数リージョンの計算資源でスループットを上げられる。データが別リージョンに出る点は、設計で確かめる。

VPC から閉域で呼ぶときは、インターフェースエンドポイント（AWS PrivateLink）を作る。エンドポイントポリシーで、呼べる操作と相手を絞れる。

モデルを自社データで調整するなら、正解付きデータを与える教師ありファインチューニングを使う。プロンプト、モデル、ナレッジベース、Lambda をつないだ固定の流れを作るなら Bedrock Flows を使う。

### 3. ナレッジベースで自社データを根拠にする

基盤モデルが答えられるのは、学習した知識の範囲だけである。社内文書の内容を答えさせたいときは、質問のたびに関連する文書を検索し、その本文をプロンプトに足してから生成させる。この手法を RAG と呼ぶ。

取り込みは、データソースの解析と分割、埋め込みモデルによるベクトル化、ベクトルストアへの保存、同期の順に進む。同期は差分だけを処理するので、2回目以降は変更のあった文書だけが対象になる。

再同期の既定はオンデマンドで、内容を変えたら自分で同期を開始する。止めたままにすると、古い内容のまま答え続ける。マネージドナレッジベースのコネクタには同期スケジュールがあり、毎日、毎週、毎月から選べる。スケジュールを設定すれば、変更が自動で反映される。

ナレッジベースには2種類ある。マネージドナレッジベースは、取り込みから検索までの基盤を Bedrock が持つ。S3、SharePoint、Confluence、Google Drive、OneDrive、Web クローラーのコネクタが用意され、文書ごとのアクセス権で検索結果を絞れる。複数の文書をたどる検索にも対応する。

カスタマーマネージドでは、ベクトルストアを自分で選んで運用する。ナレッジベースの保存先にできるのは、OpenSearch Service（マネージドクラスターと Serverless）、Aurora PostgreSQL（pgvector）、S3 Vectors、Neptune Analytics（GraphRAG）、Pinecone、Redis Enterprise Cloud、MongoDB Atlas である。RDS for PostgreSQL、MemoryDB、DocumentDB は使えず、ナレッジベースを挟まず自前で RAG を組むときの選択肢になる。

呼び出しは Retrieve と RetrieveAndGenerate に分かれる。Retrieve は検索結果だけを返すので、自分のプロンプトで加工したいときに使う。RetrieveAndGenerate は、検索から生成までを1回で済ませる。

マネージドナレッジベースはリソースポリシーを持てる。別アカウントのロールに `bedrock:Retrieve` を許可すれば、1つのナレッジベースを複数のアカウントから使える。ベクトル型のナレッジベースはリソースポリシーに対応していない。

### 4. AgentCore でエージェントに道具を使わせる

AgentCore は、エージェントの部品を分けて提供する基盤である。Strands、LangGraph、CrewAI、LlamaIndex などのフレームワークと、任意の基盤モデルを組み合わせられる。部品は単独でも使える。

| 部品 | 役割 |
|---|---|
| Runtime | エージェントをサーバーレスで動かす。セッションは分離される。 |
| Memory | 会話の短期記憶と、セッションをまたぐ長期記憶を持つ。 |
| Gateway | 既存の API や Lambda を MCP のツールに変える。 |
| Identity | Cognito などの IdP と連携し、エージェントの権限を扱う。 |
| Code Interpreter | 隔離された環境でコードを実行させる。 |
| Browser | クラウド上のブラウザで画面操作や情報取得をさせる。 |
| Observability | 実行の流れをトレースし、OpenTelemetry 形式で外部へ渡す。 |
| Registry | エージェント、ツール、MCP サーバーを社内で登録して探せるようにする。 |

以前の Bedrock Agents は Agents Classic となり、新規の受付を終えた。これから作る場合は AgentCore を使う。

### 5. SageMaker AI で自前のモデルを動かす

SageMaker AI は、機械学習モデルの学習と推論のための基盤である。推論は、入力の大きさと待てる時間で4形態から選ぶ。

| 形態 | 入力の上限 | 処理時間の目安 | 使いどころ |
|---|---|---|---|
| リアルタイム推論 | 25 MB | 応答60秒（ストリーミングは8分） | 常時動く API、低遅延が必要 |
| サーバーレス推論 | 4 MB | 60秒 | アクセスが断続的で、待機中の課金を避けたい |
| 非同期推論 | 1 GB | 60分 | 大きい入力をキューに積む。0台まで縮められる |
| バッチ変換 | GB 単位のデータセット | 数日 | 溜まったデータをまとめて処理する |

自前のモデルや独自コンテナを動かす要求は、この4形態のどれかに落ちる。

### 6. 組み込み AI で定型の処理を任せる

出力の形が決まっている処理は、基盤モデルを呼ぶより既製のサービスが安く、精度も見積もりやすい。

- Comprehend: 言語判定、感情、キーフレーズ、固有表現の抽出
- Textract: 帳票やスキャン画像からの文字と表の抽出
- Rekognition: 画像や動画の物体、顔、ラベルの検出
- Transcribe: 音声から文字への変換
- Polly: 文字から音声への変換
- Translate: 言語の翻訳
- Lex: 対話ボットの意図理解
- Personalize: 推薦
- Forecast: 時系列の需要予測（新規の受付は終了している）

指示を変えながら自由な文章を作らせたいなら、基盤モデル側に移る。

### 7. Guardrails で入出力を検査する

Guardrails は、入力と出力の両方を検査する。コンテンツフィルタは、ヘイト、侮辱、性的、暴力、不正行為、プロンプト攻撃の6分類を強度つきで判定する。拒否トピックと単語フィルタには、自社の禁止事項を足せる。個人情報は検出して伏せられる。

RAG には2つの検査が使える。文脈グラウンディング検査は、参照文書に書かれていない内容や、質問と無関係な答えを検出する。自動推論チェックは、決めた規則に反する記述を論理的に示す。

ガードレールはリクエストの引数 `guardrailIdentifier` で指定する。Bedrock 以外のモデルにも、ApplyGuardrail API で同じ検査をかけられる。

### 8. マルチアカウントに置く

モデルの利用許可は、アカウントごとに決まる。サーバーレスのモデルは商用リージョンなら既定で使え、リージョンごとに有効化する作業はない。例外は Anthropic の Claude Opus 4.7 より前のモデルで、初回利用フォームの提出が要る。組織なら管理アカウントで1回出せば、配下のアカウントに引き継がれる。AI 用のアカウントに寄せるか、各アカウントで使うかを先に決める。

寄せる場合は、インターフェースエンドポイントとエンドポイントポリシーで、呼べるアカウントと操作を絞る。ナレッジベースは、マネージド型ならリソースポリシーで共有できる。

呼び出しの記録は、モデル呼び出しログを S3 か CloudWatch Logs に出す。既定では無効で、保存先は同じアカウント、同じリージョンに限られる。

## 具体例・コード

社内文書の質問応答と、申請の自動化を1つのシステムにまとめる。

1. 社内文書を S3 に置き、マネージドナレッジベースを作る。同期スケジュールは毎日にしておく。
2. 質問は Lambda から RetrieveAndGenerate に渡す。根拠にした文書も一緒に返る。
3. ガードレールを付けて、入力の個人情報と、根拠に無い出力を弾く。
4. 申請の操作は、既存の申請 API を AgentCore Gateway でツールにする。権限は Identity 経由で渡す。
5. エージェントは AgentCore Runtime に載せ、実行のトレースを Observability から CloudWatch へ送る。
6. 夜間にまとめて回す分類処理は、バッチ推論に寄せて単価を下げる。

## つまずきポイント

- 「ナレッジベースに入れれば自動で最新になる」は間違い。同期の既定はオンデマンドで、マネージドのコネクタでスケジュールを設定しない限り、古い内容のまま残る。
- 「Bedrock があれば SageMaker AI は要らない」は間違い。自前のモデルや独自コンテナのホスティングは SageMaker AI が担う。
- 「新しいエージェントは Bedrock Agents で作る」は間違い。Agents Classic は新規受付を終えており、これからは AgentCore か、任意のフレームワークと Runtime で作る。
- 「モデルの利用許可はリージョンごとに申請する」は間違い。商用リージョンのサーバーレスのモデルは既定で使え、手動で有効化するのは GovCloud だけである。
- 「何 GB のデータでも同期呼び出しで渡せる」は間違い。リアルタイム推論には入力の上限があり、大きい入力は非同期推論かバッチ変換に回す。
- 「ガードレールを有効にすれば全リクエストに自動で効く」は間違い。リクエストで指定するか、ApplyGuardrail を自分で呼ぶ。

## 疑問・TODO

- [ ] ガードレールを複数アカウントで共有できるかを確認する
- [ ] マネージドナレッジベースの課金単位（取り込み、保存、検索）を確認する
- [ ] Amazon Q Business とナレッジベースの使い分けを整理する
- [ ] Bedrock のプロンプト管理と Flows のどちらを使うかの判断基準を書く

## 関連ノート

- [[AWS の疎結合の部品の違い]] — エージェントの裏側で使うキューとイベントの使い分け
- [[AWS のコンピューティングサービスの違い]] — Lambda や ECS にエージェントを載せる場合の比較
- [[AWS の IAM と権限の仕組み]] — モデル呼び出しとツール実行を誰に許すか
- [[AWS の暗号化と秘密情報の管理]] — プロンプトに含む秘密情報と鍵の扱い

## 出典

- https://docs.aws.amazon.com/bedrock/latest/userguide/what-is-bedrock.html
- https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base.html
- https://docs.aws.amazon.com/bedrock/latest/userguide/kb-build-managed.html
- https://docs.aws.amazon.com/bedrock/latest/userguide/kb-managed-sync.html
- https://docs.aws.amazon.com/bedrock/latest/userguide/kb-managed-cross-account.html
- https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/what-is-bedrock-agentcore.html
- https://docs.aws.amazon.com/bedrock/latest/userguide/agents.html
- https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails.html
- https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-contextual-grounding-check.html
- https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-automated-reasoning-checks.html
- https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-use-independent-api.html
- https://docs.aws.amazon.com/bedrock/latest/userguide/model-access.html
- https://docs.aws.amazon.com/bedrock/latest/userguide/cross-region-inference.html
- https://docs.aws.amazon.com/bedrock/latest/userguide/batch-inference.html
- https://docs.aws.amazon.com/bedrock/latest/userguide/prov-throughput.html
- https://docs.aws.amazon.com/bedrock/latest/userguide/model-invocation-logging.html
- https://docs.aws.amazon.com/bedrock/latest/userguide/vpc-interface-endpoints.html
- https://docs.aws.amazon.com/bedrock/latest/userguide/model-customization.html
- https://docs.aws.amazon.com/bedrock/latest/userguide/flows.html
- https://docs.aws.amazon.com/sagemaker/latest/dg/deploy-model-options.html
- https://docs.aws.amazon.com/sagemaker/latest/dg/whatis.html
- https://aws.amazon.com/bedrock/pricing/
  - https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base-setup.md
- https://docs.aws.amazon.com/prescriptive-guidance/latest/choosing-an-aws-vector-database-for-rag-use-cases/vector-db-options.html
