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
  - https://docs.aws.amazon.com/prescriptive-guidance/latest/choosing-an-aws-vector-database-for-rag-use-cases/vector-db-options.html
---

# AWS の AI サービスの使い分け

## ひとことで

モデルを借りるなら Bedrock、自前のモデルを動かすなら SageMaker AI、自社データを根拠に足すならナレッジベース、定型の抽出は既製の AI サービスになる。

## なぜ効くか / どこで使うか

分野1のタスク1.2は「AI をアーキテクチャに組み込む」設問になる。ここでの判断は、モデルを借りるか自前で動かすか、自社データを根拠に足すか、道具を使わせるかの3つに分かれる。この3つを先に決めると、登場するサービスの名前がそのまま答えになる。

逆に、どれも「AI」と一括りにして覚えると、Bedrock と SageMaker AI の役割が入れ替わる。設問が「モデルを学習から持ち込みたい」と言っているのか「既製のモデルを使いたい」と言っているのかで、正解が変わる。

## 前提となる知識

| 用語 | 一言の説明 |
|---|---|
| 基盤モデル | 大量のデータで事前学習した汎用のモデル。文章生成や画像生成の土台になる。 |
| 推論 | 学習済みのモデルに入力して、出力を得ること。 |
| トークン | モデルが入出力を数える単位。課金の単位でもある。 |
| 埋め込み | 文章を数値の並び（ベクトル）に変えること。意味の近さを距離で測れる。 |
| ベクトルストア | 埋め込みを保存して、近いものを検索できるデータベース。 |
| RAG | 検索して得た情報をプロンプトに足してから生成する手法。 |
| エージェント | モデルが道具（API）を選んで実行し、目的の作業を進める仕組み。 |
| ツール呼び出し | モデルが外部の関数や API を呼ぶ出力を返す機能。 |
| ナレッジベース | データソース、埋め込み、ベクトルストア、検索をまとめた単位。 |
| ガードレール | 入力と出力を検査して、危険な内容や個人情報を止める仕組み。 |

## 本体の解説

### 1. 3つの選択で分ける

| 選ぶもの | 使うサービス | 向く場面 |
|---|---|---|
| モデルを借りる | Amazon Bedrock | 既製の基盤モデルを API で使いたい |
| 自前のモデルを動かす | SageMaker AI のエンドポイント | 学習済みモデルや独自コンテナを持ち込みたい |
| 定型の処理を任せる | 組み込み AI（Comprehend など） | 分類、抽出、文字起こしなど出力の形が決まっている |
| 自社データを根拠にする | Bedrock ナレッジベース | 社内文書を根拠に答えさせたい |
| 道具を使わせる | Bedrock AgentCore | 申請や検索などの操作をモデルに実行させたい |
| 入出力を検査する | Bedrock Guardrails | 不適切な内容や個人情報を止めたい |

### 2. Bedrock（モデルを借りる）

Amazon Bedrock は、複数の提供元の基盤モデルを API 経由で使うフルマネージドのサービスである。100以上のモデルが選べ、サーバーや GPU の管理は持たない。呼び出しは `bedrock-runtime` の Converse（モデル間の差分を吸収する共通形式）か InvokeModel（モデル固有の形式）で行う。

課金はトークン単位になる。大量の入力をまとめて処理するバッチ推論は、オンデマンドより50%安い。スループットを固定したいときは Provisioned Throughput を使い、モデルユニット単位で購入する。コミットなし、1か月、6か月の3つから選び、長いほど時間単価が下がる。

モデルの利用許可は、アカウントとリージョンの組ごとに有効化する。多くは AWS Marketplace の権限があれば最初から使える状態になっている。リージョンをまたいで処理させる推論プロファイル（地理単位とグローバル）もあり、スループットを上げられるが、データが別リージョンに出る点は設計で確認する。

閉域にしたいときは VPC インターフェースエンドポイント（AWS PrivateLink）を使う。`bedrock-runtime` などのエンドポイントを VPC 内に作り、エンドポイントポリシーで呼べる操作と相手を絞る。

モデルの調整は、正解付きのデータを与える教師ありファインチューニングがある。処理の流れを組むなら Bedrock Flows で、プロンプト、モデル、ナレッジベース、Lambda をつないで固定のワークフローにする。

### 3. ナレッジベース（自社データを根拠にする）

基盤モデルは学習した知識しか持たない。社内文書の内容を答えさせたいときは、質問のたびに関連する文書を検索し、その本文をプロンプトに足してから生成する。これが RAG である。

取り込みは、データソースを解析して分割し、埋め込みモデルでベクトルにして、ベクトルストアに保存する流れになる。データソースを変更したら再同期が要る。同期しないと、古い内容のまま答え続ける。

ナレッジベースには2種類ある。マネージドナレッジベースは、取り込み、索引、保存、検索の基盤を Bedrock が持つ。コネクタが S3、SharePoint、Confluence、Google Drive、OneDrive、Web クローラに用意されていて、文書ごとのアクセス権で絞り込む機能もある。複数の文書をたどる検索（マルチホップ）にも対応する。カスタマーマネージドは、ベクトルストアを自分で選んで運用する。OpenSearch Service、Aurora や RDS の PostgreSQL（pgvector）、Neptune Analytics、MemoryDB、DocumentDB、S3 Vectors が候補になる。

呼び出しは Retrieve（検索だけ）と RetrieveAndGenerate（検索して生成まで）の2つに分かれる。検索結果を自分のプロンプトで加工したいなら前者、答えまで欲しいなら後者になる。

マネージドナレッジベースはリソースポリシーを持てる。別アカウントのロールに `bedrock:Retrieve` を許可すれば、ナレッジベースを1つに集めて複数アカウントから使わせられる。ベクトル型のナレッジベースはリソースポリシーに対応していない。

### 4. AgentCore（道具を使わせる）

Amazon Bedrock AgentCore は、エージェントを動かすための部品を分けて提供する基盤である。Strands、LangGraph、CrewAI、LlamaIndex などのフレームワークと、任意の基盤モデルを組み合わせられる。部品は単独でも使える。

| 部品 | 役割 |
|---|---|
| Runtime | エージェントをサーバーレスで動かす。セッションは分離される。 |
| Memory | 会話の短期記憶と、セッションをまたぐ長期記憶を持つ。 |
| Gateway | 既存の API や Lambda を MCP のツールに変えて、エージェントから呼べるようにする。 |
| Identity | Cognito などの IdP と連携して、エージェントの権限と認証を扱う。 |
| Code Interpreter | 隔離された環境でコードを実行させる。 |
| Browser | クラウド上のブラウザで画面操作や情報取得をさせる。 |
| Observability | 実行の流れをトレースし、OpenTelemetry 形式で外部の監視に渡す。 |
| Registry | エージェント、ツール、MCP サーバーを社内で登録して探せるようにする。 |

以前の Bedrock Agents は Amazon Bedrock Agents Classic という扱いになり、新規の受付を終了した。これから作るなら AgentCore 側になる。

### 5. SageMaker AI（自前のモデルを動かす）

SageMaker AI は、機械学習モデルの学習と推論のための基盤である。推論は入力の大きさと待てる時間で4形態から選ぶ。

| 形態 | 入力の上限 | 処理時間の目安 | 使いどころ |
|---|---|---|---|
| リアルタイム推論 | 25 MB | 60秒（ストリーミングは8分） | 常時動く API、低遅延が必要 |
| サーバーレス推論 | 4 MB | 60秒 | アクセスが断続的で、待機中の課金を避けたい |
| 非同期推論 | 1 GB | 60分 | 大きい入力をキューに積んで処理する。0台まで縮められる |
| バッチ変換 | GB 単位のデータセット | 数日 | 溜まったデータをまとめて処理する |

自前のモデルや独自コンテナをホスティングする要求は、この4形態のどれかに落ちる。Bedrock はモデルを借りる前提なので、ここが分かれ目になる。

### 6. 組み込み AI（定型の処理を任せる）

出力の形が決まっている処理は、基盤モデルを呼ぶより既製のサービスが安く、精度も読みやすい。

- Comprehend: 文章の言語判定、感情、キーフレーズ、固有表現の抽出
- Textract: 帳票やスキャン画像からの文字と表の抽出
- Rekognition: 画像や動画の物体、顔、ラベルの検出
- Transcribe: 音声から文字への変換
- Polly: 文字から音声への変換
- Translate: 言語の翻訳
- Lex: 対話ボットの意図理解
- Personalize: 推薦
- Forecast: 時系列の需要予測

自由な文章を作らせたい、指示を変えながら使いたい、という要求が出たら基盤モデル側に移る。

### 7. 出力を検査する（Guardrails）

Amazon Bedrock Guardrails は、入力と出力を検査する。コンテンツフィルタは、ヘイト、侮辱、性的、暴力、不正行為、プロンプト攻撃の6分類を強度つきで判定する。拒否トピックと単語フィルタは、自社の禁止事項を足せる。個人情報は検出して伏せられる。

RAG と組み合わせる検査も2つある。文脈グラウンディング検査は、参照した文書に書かれていない内容や、質問と無関係な答えを検出する。自動推論チェックは、決めた規則に反する記述を論理的に示す。

ガードレールはリクエストの引数（`guardrailIdentifier`）で指定する。Bedrock 以外のモデルにも、ApplyGuardrail API で同じ検査をかけられる。

### 8. マルチアカウントに置くとき

モデルの利用許可はアカウントとリージョンの組で決まるため、1つのアカウントで有効化しても他には広がらない。AI 用のアカウントに寄せるか、各アカウントで有効化するかを決める。

寄せる場合は、VPC インターフェースエンドポイントとエンドポイントポリシーで、呼べるアカウントと操作を絞る。ナレッジベースは、マネージド型ならリソースポリシーで共有できる。

呼び出しの記録は、Bedrock のモデル呼び出しログを S3 か CloudWatch Logs に出す。既定では無効で、保存先は同じアカウント・同じリージョンに限られる。

## 具体例・コード

社内文書の質問応答と、申請の自動化を1つにまとめる場合の流れ。

1. 社内文書を S3 に置き、マネージドナレッジベースを作って同期する。
2. 質問は Lambda から RetrieveAndGenerate に渡す。根拠にした文書も一緒に返る。
3. ガードレールを付けて、入力の個人情報と、根拠に無い出力を弾く。
4. 申請の操作は、既存の申請 API を AgentCore Gateway でツールにする。権限は Identity 経由で渡す。
5. エージェントは AgentCore Runtime に載せ、実行のトレースを Observability から CloudWatch に送る。
6. 夜間にまとめて回す分類処理は、バッチ推論に寄せて単価を下げる。

## つまずきポイント

- ナレッジベースに入れれば自動で最新になると思う。実際は、データソースを変えたあとに再同期が要る。
- Bedrock があれば SageMaker AI は不要だと思う。実際は、自前のモデルや独自コンテナをホスティングするのは SageMaker AI の仕事になる。
- 新しいエージェントは Bedrock Agents で作ると思う。実際は Agents Classic として新規受付が終わっていて、AgentCore か任意のフレームワークと Runtime の組み合わせになる。
- モデルの利用許可はリージョンで1回取れば足りると思う。実際は、アカウントとリージョンの組ごとに要る。
- 何 GB のデータでも同期呼び出しで渡せると思う。実際はリアルタイム推論に上限があり、大きい入力は非同期推論かバッチ変換に回す。
- ガードレールを有効にすれば全リクエストに自動で効くと思う。実際は、リクエストで指定するか、ApplyGuardrail を自分で呼ぶ。

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
- https://docs.aws.amazon.com/prescriptive-guidance/latest/choosing-an-aws-vector-database-for-rag-use-cases/vector-db-options.html
