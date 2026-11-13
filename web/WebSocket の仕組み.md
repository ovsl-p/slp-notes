---
title: WebSocket の仕組み
aliases: [WebSocket の仕組み, WebSocket, websocket, ws, wss, WebSocket プロトコル, RFC 6455]
domain: web
tags: [websocket, http, tcp, realtime, protocol, rfc6455, framing, browser]
created: 2026-11-13
updated: 2026-11-13
status: growing
prerequisites: [TCP とソケット, HTTP/1.1 のメッセージ形式, ステータスコード 101, バイトオーダー, base64 と SHA-1, フレーミング, 全二重通信とバックプレッシャー, TLS, イベントループと非同期 I/O]
related: []
sources:
  - RFC 6455, The WebSocket Protocol
  - RFC 7692, Compression Extensions for WebSocket (permessage-deflate)
  - RFC 8441, Bootstrapping WebSockets with HTTP/2
  - MDN Web Docs, WebSocket API
  - WHATWG HTML Living Standard, Web sockets
---

# WebSocket の仕組み

## ひとことで

HTTP の接続をそのまま使い続けて、双方向にメッセージを送れるようにしたプロトコル。最初のハンドシェイクだけ HTTP、その後は独自のフレームで通信する。

## なぜ効くか / どこで使うか

HTTP は「クライアントが聞いたらサーバが答える」一方通行。サーバから能動的に何かを送りたくなると、これが邪魔になる。

- チャット、通知、コラボ編集、オンラインゲーム、株価／ダッシュボード、プレゼンス（誰がオンラインか）
- サーバ→クライアントの「今起きた」を、リクエストを待たずに届けたい場面すべて
- IoT 機器の遠隔操作、ログのストリーミング

代替手段との比較：

- **ポーリング**: 一定間隔で GET。無駄が多く、遅延も間隔ぶん
- **ロングポーリング**: レスポンスを保留して擬似的に push。1メッセージごとに再接続が要る
- **SSE（Server-Sent Events）**: サーバ→クライアントは得意だが、クライアント→サーバは別リクエスト。HTTP/1.1 では接続数制限にも当たる
- **WebSocket**: 1本の TCP 接続で双方向。オーバーヘッドが小さく、低遅延
- **WebTransport / WebRTC**: UDP ベースでさらに低遅延。音声・映像や信頼性不要の通信向け

WebSocket が向くのは「**双方向で、順序と到達を保証したい、テキストか小さめのバイナリ**」ケース。TCP の上なので、順序保証はあるが**再送による詰まり（head-of-line blocking）**は避けられない。

## 前提となる知識

| 知っておくこと | なぜ必要か |
| --- | --- |
| TCP とソケット | WebSocket は TCP の上のプロトコル。接続は1本のソケットで、閉じれば終わり |
| HTTP/1.1 のメッセージ形式 | ハンドシェイクは HTTP リクエストそのもの。ヘッダが読めないと何が起きているか分からない |
| ステータスコード 101 | `101 Switching Protocols` が「ここからプロトコルを切り替える」の合図 |
| `Upgrade` / `Connection` ヘッダ | HTTP の上で別プロトコルに乗り換える仕組み（HTTP/1.1 の hop-by-hop ヘッダ） |
| base64 と SHA-1 | ハンドシェイクの受け入れ確認（`Sec-WebSocket-Accept`）に使う。暗号学的な強度のためではない |
| バイトオーダー | フレームの長さはネットワークバイトオーダー（ビッグエンディアン）で書かれる |
| フレーミング | 「どこからどこまでが1メッセージか」を長さプレフィクスで表す考え方。TCP はストリームなので必須 |
| 全二重とバックプレッシャー | 送る側と受ける側の速度差をどう扱うか。ここを設計しないとメモリが膨らむ |
| TLS | `wss://` は TLS の上の WebSocket。混在コンテンツとしてブロックされる点も |
| イベントループと非同期 I/O | サーバ側で何万接続も捌くには、1接続1スレッドでは足りない |

## 本体の解説

### 1. 全体像：HTTP で始まり、HTTP で終わらない

```
クライアント                                サーバ
    |--- HTTP GET (Upgrade: websocket) ----->|   ← ここまで HTTP
    |<-- 101 Switching Protocols ------------|   ← ここでプロトコル切替
    |                                        |
    |<======= WebSocket フレーム ===========>|   ← 以後は独自フレーム
    |<======= （双方向・全二重） ============>|
    |--- Close フレーム -------------------->|
```

ポイントは **接続は1本のまま**であること。HTTP のようにリクエストごとに接続を張り直さない（HTTP/1.1 の keep-alive に近いが、やり取りの主導権が双方向になる）。

### 2. ハンドシェイク（実物）

クライアントが送るリクエスト：

```http
GET /chat HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
Sec-WebSocket-Protocol: chat, superchat
Origin: https://example.com
```

- `Sec-WebSocket-Key` は **16バイトのランダム値を base64 したもの**（この例は RFC のサンプル）
- `Sec-WebSocket-Version: 13` が現行版（13 以外は 426 Upgrade Required で返す）
- `Sec-WebSocket-Protocol` はアプリ独自のサブプロトコル名の候補
- `Origin` はブラウザが自動で付ける。**サーバ側で必ず検証する**（後述）

サーバの応答：

```http
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
Sec-WebSocket-Protocol: chat
```

`Sec-WebSocket-Accept` は次の計算の結果：

```
accept = base64( SHA1( key + "258EAFA5-E914-47DA-95CA-C5AB0DC85B11" ) )
```

あの謎の GUID は **「HTTP のキャッシュやプロキシが偶然このハンドシェイクを成功させてしまう」のを防ぐための固定文字列**。クライアントが送った Key をそのまま返さず、GUID を混ぜてハッシュ化させることで、WebSocket を理解していない中間装置が誤って 101 を返すことを難しくしている。

手元で確認した値：

```
key    = dGhlIHNhbXBsZSBub25jZQ==
accept = s3pPLMBiTxaQ9kYGzzhZRbK+xOo=   ← RFC 6455 の記載と一致
```

### 3. フレーム形式

ハンドシェイク後は、すべてのデータが「フレーム」で運ばれる。

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-------+-+-------------+-------------------------------+
|F|R|R|R| opcode|M| Payload len |    Extended payload length    |
|I|S|S|S|  (4)  |A|     (7)     |        (16 / 64 bit)          |
|N|V|V|V|       |S|             |                               |
| |1|2|3|       |K|             |                               |
+-+-+-+-+-------+-+-------------+-------------------------------+
|                         Masking-key (4 bytes, if MASK=1)      |
+---------------------------------------------------------------+
|                         Payload Data ...                      |
+---------------------------------------------------------------+
```

- **FIN (1bit)**: このフレームでメッセージが完結するか
- **RSV1-3 (3bit)**: 拡張用。予約。拡張を使わないなら 0
- **opcode (4bit)**: フレームの種類（下記）
- **MASK (1bit)**: マスクされているか
- **Payload len (7bit)**: 0〜125 はそのまま。**126** なら続く 2 バイト（16bit）、**127** なら続く 8 バイト（64bit）が本当の長さ
- **Masking-key (4byte)**: MASK=1 のときだけ存在

opcode：

| opcode | 意味 |
| --- | --- |
| 0x0 | continuation（分割メッセージの続き） |
| 0x1 | text（UTF-8） |
| 0x2 | binary |
| 0x8 | close |
| 0x9 | ping |
| 0xA | pong |

**制御フレーム（0x8〜0xA）は 125 バイト以下で、分割できない**。ping/pong はアプリのデータではないので、間に挟まっても順序を乱さない。

### 4. マスキング：なぜクライアントだけ必須なのか

- **クライアント → サーバ: 必ずマスクする**
- **サーバ → クライアント: マスクしてはいけない**（したら切断）

マスクは「4バイトの鍵を繰り返し XOR する」だけの弱い難読化で、暗号ではない。目的は**プロキシ汚染攻撃（cache poisoning）の防止**。悪意あるページが、プロキシが HTTP として解釈しうるバイト列を WebSocket フレームとして送り込む、という攻撃を、フレームごとにランダムなマスクで潰している。

実装では「受信時に MASK ビットを見て、立っていたら鍵を読んで XOR を外す」だけ。

### 5. メッセージと分割（fragmentation）

1つのアプリケーションメッセージは、複数フレームに分割できる。

```
[text, FIN=0] → [continuation, FIN=0] → [continuation, FIN=1]
```

- 受信側は FIN=1 まで結合してから1メッセージとして扱う
- 巨大メッセージを一気に送らずに済むが、**実装によっては上限がある**（多くのサーバは 1MB などで切る）
- 制御フレームは分割の途中に挟まってもよい（割り込み可能）

### 6. 接続の終わり方

正常終了は **close ハンドシェイク**：片方が `close` フレーム（ステータスコード + 理由）を送り、相手が `close` を返して TCP を閉じる。

主要なステータスコード：

| code | 意味 |
| --- | --- |
| 1000 | 正常終了 |
| 1001 | 相手が去る（ページ遷移など） |
| 1002 | プロトコルエラー |
| 1003 | 受け取れないデータ型 |
| 1008 | ポリシー違反 |
| 1009 | メッセージが大きすぎる |
| 1011 | サーバ内部エラー |
| 1006 | **異常終了（このコードは送れない。切断を検知した側がローカルで使う）** |

`1006` は「close フレームを受け取らずに TCP が切れた」ことを表す**ローカルな値**。ネットワーク断・プロキシのタイムアウト・サーバの強制終了などで出る。**アプリ側は 1006 を見たら再接続を検討する**。

### 7. 拡張とサブプロトコル

- **サブプロトコル**（`Sec-WebSocket-Protocol`）: `chat` とか `graphql-ws` など、アプリ層のプロトコル名を合意する。サーバが候補から1つ選んで返す
- **拡張**（`Sec-WebSocket-Extensions`): フレームの圧縮など。代表は **permessage-deflate**（RFC 7692）。CPU とレイテンシのトレードオフがあるので、小さいメッセージが多い場合は逆効果になりうる

### 8. HTTP/2・HTTP/3 との関係

- WebSocket は **HTTP/1.1 の Upgrade** が前提。HTTP/2 の通常のストリームでは Upgrade が使えない
- そのため HTTP/2 上で WebSocket を使うには **RFC 8441（Extended CONNECT, `:protocol = websocket`）** が必要。対応は実装依存
- HTTP/3 の上では WebTransport が本命（WebSocket とは別物）

「HTTP/2 だから WebSocket が速くなる」わけではない。**接続の多重化と WebSocket は別の話**。

### 9. セキュリティ

- **`Origin` をサーバで検証する**。ブラウザは Cookie を付けてハンドシェイクを送るので、検証しないと**クロスサイト WebSocket ハイジャック**（CSRF の WebSocket 版）になる
- **`wss://` を使う**。https のページから `ws://` は混在コンテンツとしてブロックされる
- **認証はハンドシェイク時**（Cookie / `Authorization` ヘッダ / クエリのチケット）に済ませる。`Sec-WebSocket-Protocol` にトークンを載せる小技もある
- **認可はメッセージ単位で**。接続時に「誰か」を確定しても、「この操作をしていいか」は毎回見る
- **トークンを URL のクエリに置くのは避ける**（アクセスログ・Referer に残る）

### 10. スケーリングの実務

- WebSocket は**ステートフル**。ロードバランサは接続を特定ノードに固定する必要がある（スティッキーセッション、または L4 バランシング）
- 複数ノードで全員に配信したいなら **pub/sub**（Redis / NATS / Kafka）を挟み、各ノードが購読して自分の接続に流す
- **アイドルタイムアウト**に注意。プロキシや LB が数十秒〜数分で切るので、**ping/pong で心拍**を打つ（サーバ主導。ブラウザの JS API からは ping を送れない）
- **再接続は自前で実装**する。指数バックオフ + ジッタ、切断中のメッセージはキューに積む
- **バックプレッシャー**: 送る速度が回線を超えるとバッファが膨らむ。ブラウザは `bufferedAmount` しか見られないので、アプリ側で「ACK が返るまで次の塊を送らない」などの制御を入れる

## 具体例・コード

### 最小のサーバ（Python 標準ライブラリのみ）

ハンドシェイク → 1フレーム受信 → エコー返送。**手元で実際に動かして、ブラウザ相当の生ソケットクライアントと往復できることを確認済み**。

```python
import base64, hashlib, socket, struct

GUID = "258EAFA5-E914-47DA-95CA-C5AB0DC85B11"

def accept_key(key: str) -> str:
    return base64.b64encode(hashlib.sha1((key + GUID).encode()).digest()).decode()

def parse_frame(sock):
    def recvn(k):
        buf = b""
        while len(buf) < k:
            c = sock.recv(k - len(buf))
            if not c:
                raise ConnectionError("closed")
            buf += c
        return buf
    h = recvn(2)
    fin, op = h[0] & 0x80, h[0] & 0x0F
    masked, ln = h[1] & 0x80, h[1] & 0x7F
    if ln == 126:
        ln = struct.unpack(">H", recvn(2))[0]
    elif ln == 127:
        ln = struct.unpack(">Q", recvn(8))[0]
    key = recvn(4) if masked else None
    data = recvn(ln)
    if masked:
        data = bytes(x ^ key[i % 4] for i, x in enumerate(data))
    return fin, op, data

def serve(sock):
    req = b""
    while b"\r\n\r\n" not in req:
        req += sock.recv(4096)
    headers = {}
    for line in req.decode().split("\r\n")[1:]:
        if ":" in line:
            k, v = line.split(":", 1)
            headers[k.strip().lower()] = v.strip()
    key = headers["sec-websocket-key"]
    sock.sendall(("HTTP/1.1 101 Switching Protocols\r\n"
                  "Upgrade: websocket\r\nConnection: Upgrade\r\n"
                  f"Sec-WebSocket-Accept: {accept_key(key)}\r\n\r\n").encode())
    fin, op, data = parse_frame(sock)          # クライアントの1メッセージ
    payload = b"echo:" + data
    sock.sendall(bytes([0x80 | 0x1, len(payload)]) + payload)   # サーバは非マスク
    sock.close()
```

実際のログ：

```
クライアント: HTTP/1.1 101 Switching Protocols
             Sec-WebSocket-Accept: BACScCJPNqyz+UBoqMH89VmURoA=
サーバ受信:   b'hello ws'  opcode 1  fin True
クライアント: (fin=128, opcode=1, b'echo:hello ws')
```

### ブラウザ側

```js
const ws = new WebSocket("wss://example.com/chat");   // サブプロトコルは第2引数

ws.addEventListener("open", () => ws.send("hello"));
ws.addEventListener("message", (e) => {
  // e.data は string（text）か Blob/ArrayBuffer（binary）
  console.log(e.data);
});
ws.addEventListener("close", (e) => {
  console.log(e.code, e.reason);   // 1006 なら異常切断
});
ws.addEventListener("error", () => {});   // 詳細は分からない。close で判断

// 状態: CONNECTING(0) / OPEN(1) / CLOSING(2) / CLOSED(3)
```

### 再接続（指数バックオフ）

```js
let retry = 0;
function connect() {
  const ws = new WebSocket("wss://example.com/chat");
  ws.onopen = () => { retry = 0; };
  ws.onclose = (e) => {
    if (e.code === 1000) return;                 // 正常終了なら再接続しない
    const delay = Math.min(30000, 500 * 2 ** retry++);
    setTimeout(connect, delay + Math.random() * 1000);   // ジッタで一斉再接続を散らす
  };
}
connect();
```

## つまずきポイント

- **`ws://` と `wss://` を混ぜる**。https ページから `ws://` はブロックされる。本番は常に `wss`
- **プロキシ／LB の Upgrade 未対応**。企業プロキシや古い LB は `Upgrade` を落とす。ハンドシェイクが 400/502 で落ちるならここを疑う
- **アイドルタイムアウトで切られる**。ping/pong かアプリ層の heartbeat を入れないと、静かな時間のあと突然 1006
- **再接続を実装していない**。ブラウザ標準 API には自動再接続がない
- **`bufferedAmount` を見ていない**。大量送信でメモリが膨らみ、やがて切断される
- **`error` イベントから原因が分からない**。詳細は `close` のコードとサーバログで見る
- **ブラウザから ping は送れない**。JS API に ping は無い。keepalive はサーバ主導か、アプリ層のメッセージでやる
- **メッセージサイズ上限**。サーバ実装ごとに既定値がある（1MB など）。大きいペイロードは分割や別経路に
- **順序は保証されるが高速化はされない**。1本の TCP なので、1つの大きなメッセージが後続を詰まらせる（head-of-line blocking）
- **モバイルのバックグラウンド**で切られる。復帰時に再接続＋状態の再同期が必要
- **`Origin` 検証を忘れる**。Cookie 認証と組み合わせると、他サイトから接続される
- **認可を接続時だけやる**。「ログイン済みだから全部OK」は NG。操作ごとに権限を見る
- **1接続1スレッドで書く**。数千接続で破綻する。イベントループ（Node / asyncio / Go / Rust の非同期）で

## 疑問・TODO

- [ ] permessage-deflate を有効にしたときの CPU とレイテンシを実測（小メッセージ多め／大きめで比較）
- [ ] RFC 8441（HTTP/2 上の WebSocket）の対応状況を主要サーバで確認
- [ ] ブラウザの同時接続数上限（HTTP/1.1 の 6 本制限）と WebSocket の関係を整理
- [ ] 10万接続を想定したときの OS 側チューニング（ulimit、epoll、ソケットバッファ）
- [ ] WebTransport との使い分け（音声・ゲームでの実測）

## 関連ノート

- WebTransport と WebRTC の違い（未作成）
- HTTP/2 / HTTP/3 の基礎（未作成）
- TCP とソケットの基礎（未作成）

## 出典

- RFC 6455, *The WebSocket Protocol*: https://www.rfc-editor.org/rfc/rfc6455
- RFC 7692, *Compression Extensions for WebSocket*: https://www.rfc-editor.org/rfc/rfc7692
- RFC 8441, *Bootstrapping WebSockets with HTTP/2*: https://www.rfc-editor.org/rfc/rfc8441
- MDN Web Docs, *WebSocket API*: https://developer.mozilla.org/docs/Web/API/WebSockets_API
- WHATWG HTML Living Standard, *Web sockets*: https://html.spec.whatwg.org/multipage/web-sockets.html
