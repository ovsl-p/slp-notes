---
title: Kubernetes の概要
aliases: [Kubernetes の概要, Kubernetes, k8s, クバネティス, コンテナオーケストレーション, kubectl]
domain: web
tags: [kubernetes, container, orchestration, infra, declarative, devops]
created: 2026-11-14
updated: 2026-11-14
status: growing
prerequisites: [コンテナとイメージ, Linux の namespace と cgroups, IP と DNS とポート, プロセスとシグナル, 宣言的設定と制御ループ, ロードバランシングの基礎]
related: ["[[WebSocket の仕組み]]"]
sources:
  - Kubernetes Documentation（https://kubernetes.io/docs/）
  - Kubernetes API Reference（https://kubernetes.io/docs/reference/kubernetes-api/）
  - "Kubernetes Up & Running（Burns, Beda, Hightower）"
---

# Kubernetes の概要

## ひとことで

コンテナを「こうなっていてほしい」と宣言すると、クラスタがその状態を保ち続けてくれる仕組み。

## なぜ効くか / どこで使うか

コンテナを1台のマシンで動かすのは簡単だが、本番では次の問題が一気に出てくる。

- 落ちたら誰が再起動するのか
- デプロイ中に無停止で入れ替えられるか
- 負荷が上がったら増やせるか
- どのコンテナがどこで動いているかを、どうやって互いに見つけるのか

Kubernetes はこれを**宣言的な設定ファイル**で扱えるようにする。人間が手順を書くのではなく「最終的にこうなっていてほしい」を書き、差分を埋めるのはクラスタの仕事。

## 前提となる知識

| 知っておくこと | なぜ必要か |
| --- | --- |
| コンテナとイメージ | Kubernetes が動かす対象。イメージ = 実行環境のスナップショット、コンテナ = その実行 |
| Linux の namespace と cgroups | コンテナの隔離とリソース制限の実体。Kubernetes はその上に乗っているだけ |
| IP・DNS・ポート | Pod 間通信、Service の名前解決、Ingress の振り分けが全部ここ |
| プロセスとシグナル | 停止時に SIGTERM を送って猶予後に SIGKILL。graceful shutdown の設計に効く |
| 宣言的設定と制御ループ | Kubernetes の中心思想。「あるべき状態」と「今の状態」を比べて差分を埋める |
| ロードバランシングの基礎 | Service が何をしているかの理解に必要 |

## 本体の解説

### 1. 全体像

```
┌──────────── コントロールプレーン ────────────┐
│  kube-apiserver   ← すべての入口（REST API）   │
│  etcd             ← クラスタの状態を保存       │
│  kube-scheduler   ← Pod をどのノードに置くか   │
│  controller-manager ← 差分を埋める制御ループ群 │
└───────────────────────────────────────────────┘
                    ↕
┌──────────────── ノード（ワーカー） ───────────┐
│  kubelet          ← Pod を実際に動かす         │
│  container runtime ← コンテナ実行（containerd 等）│
│  kube-proxy / CNI ← ネットワークと Service     │
│  Pod Pod Pod ...                              │
└───────────────────────────────────────────────┘
```

**api-server が唯一の入口**。`kubectl` もコントローラも kubelet も、全部 api-server と会話する。状態は **etcd** に保存され、複数ノードで合意を取る（Raft）。

### 2. 宣言的 + 制御ループ（reconciliation）

Kubernetes の本質はこれだけ。

1. あなたが「Pod を3つ維持して」と書く（desired state）
2. etcd に保存される
3. コントローラが定期的に「今の状態」と比べる
4. 足りなければ作る、多ければ消す

だから「落ちても勝手に戻る」。誰かが手で消しても、次のループで復活する。**命令ではなく状態を書く**のがポイント。

### 3. Pod が最小単位

コンテナを直接ではなく、**Pod** という箱に入れて動かす。Pod 内のコンテナは次を共有する。

- ネットワーク（同じ IP、`localhost` で会話できる）
- ボリューム
- ライフサイクル（一緒に作られ、一緒に消える）

なぜこの単位かというと、「補助的な処理を同じマシン・同じネットワークで動かしたい」ユースケースが多いから（ログ転送、プロキシ、初期化処理などのサイドカー）。

Pod は**使い捨て**。再起動すると IP も変わる。だから Pod を直接運用せず、上位のオブジェクトに管理させる。

### 4. Deployment → ReplicaSet → Pod

```
Deployment  （「このイメージを3つ、こう更新して」）
   └ ReplicaSet （「3つ維持する」）
        └ Pod, Pod, Pod
```

- **Deployment**: ローリング更新やロールバックを扱う
- **ReplicaSet**: 指定数の Pod を維持する
- **Pod**: 実際の実行単位

ローリング更新は「新しい ReplicaSet を少しずつ増やし、古いのを減らす」ことで実現する。`maxSurge` / `maxUnavailable` で速度と可用性のバランスを決める。

### 5. Service と Ingress

Pod の IP は変わるので、**変わらない入口**が必要になる。それが Service。

| 種類 | 役割 |
| --- | --- |
| ClusterIP | クラスタ内だけで名前解決できる仮想 IP（既定） |
| NodePort | 各ノードのポートで外から入れる |
| LoadBalancer | クラウドの LB を払い出す |
| ExternalName | DNS のエイリアス |

Service は **ラベルセレクタ**で Pod を選ぶ。IP ではなくラベルで結びつくので、Pod が入れ替わっても追従する。内部 DNS で `my-svc.my-namespace.svc.cluster.local` のように引ける。

**Ingress** は HTTP のルーティング（パスやホスト名で振り分け）。ただし Ingress リソース自体は「設定の宣言」で、実際に動かすには **Ingress controller**（nginx、Traefik、AWS Load Balancer Controller など）が別途必要。

### 6. 設定と秘密

- **ConfigMap**: 設定値（環境変数やファイルとして注入）
- **Secret**: パスワードやトークン。base64 なだけで暗号化ではないので、etcd の暗号化や外部の秘密管理（Vault、Sealed Secrets など）と組み合わせる

### 7. スケジューリングとリソース

- **requests**: この Pod には最低これだけ CPU/メモリを確保してね（スケジューラの判断材料）
- **limits**: これ以上は使わせない（CPU は throttle、メモリは OOMKill）

`requests` を書かないと、ノードが詰まってから OOM や Evicted で落ちる。**最初に書くべきは requests**。

他に taints/tolerations（ノード側から排除）、affinity（Pod を寄せる/離す）など。

### 8. ストレージとステートフル

- **Volume**: Pod にマウントする記憶領域（emptyDir は Pod の寿命と同じ）
- **PersistentVolume / PVC**: ストレージを抽象化して要求する
- **StatefulSet**: データベースのように「名前とディスクが固定」の Pod を扱う

ステートフルなものは Kubernetes が苦手な領域。マネージド DB を使う判断も普通。

### 9. ヘルスチェック

- **liveness**: 死んでいたら再起動
- **readiness**: 準備できていなければ Service の宛先から外す
- **startup**: 起動が遅いアプリ用（これが通るまで liveness は待つ）

readiness を書かないと、起動途中の Pod にリクエストが飛んで 502 になる。

### 10. よくある誤解

- **Kubernetes は Docker の代替ではない**。コンテナを動かすランタイム（containerd など）の上で、それらを管理する層
- **Pod は消えるもの**。IP も名前も固定されない。状態を持つなら外に出す
- **YAML を書けば動くわけではない**。requests、probe、Service のセレクタが揃って初めて実用的になる
- **小規模なら過剰**。1台で足りるなら Docker Compose でいい

## 具体例・コード

最小の Deployment + Service。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  selector:
    matchLabels: { app: web }
  template:
    metadata:
      labels: { app: web }
    spec:
      containers:
        - name: web
          image: nginx:1.27
          ports: [{ containerPort: 80 }]
          resources:
            requests: { cpu: 100m, memory: 128Mi }
            limits:   { memory: 256Mi }
          readinessProbe:
            httpGet: { path: /, port: 80 }
            initialDelaySeconds: 3
---
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector: { app: web }     # ← Deployment のラベルと一致させる
  ports:
    - port: 80
      targetPort: 80
```

```bash
kubectl apply -f web.yaml
kubectl get pods -o wide
kubectl describe pod <name>        # イベントとエラーはここ
kubectl logs <name>
kubectl exec -it <name> -- sh
kubectl rollout status deploy/web
kubectl rollout undo deploy/web
```

ローカルで試すなら kind、minikube、k3d のどれか。1ノードのクラスタが数分で立つ。

## つまずきポイント

- **`CrashLoopBackOff`**: 起動して落ちるを繰り返している。`kubectl logs --previous` で前回のログを見る
- **`ImagePullBackOff`**: イメージ名の誤り、レジストリの認証、ネットワークの遮断
- **Service に繋がらない**: セレクタのラベル不一致、`targetPort` の誤り、readiness が通っていない。`kubectl get endpoints <svc>` が空ならまずラベル
- **Pod が Pending**: リソース不足（requests が大きすぎる）、taints、PVC がバインドされない
- **OOMKilled**: limits を超えた。メモリの実測値を見てから決める
- **`kubectl apply` したのに反映されない**: namespace の指定漏れ（`-n`）、immutable なフィールドの変更
- **Ingress を作ったのに 404**: Ingress controller が入っていない、class の指定漏れ
- **全部 latest タグ**: どのバージョンが動いているか分からなくなる。タグは固定する

## 疑問・TODO

- [ ] kind で1ノードクラスタを立てて、Deployment のローリング更新を観察する
- [ ] readiness を外したとき、更新中にどれくらい 502 が出るか測る
- [ ] HPA（自動スケール）が効く条件を、CPU とレイテンシで比較する
- [ ] eBPF ベースの CNI（Cilium）と kube-proxy の違いを整理する
- [ ] StatefulSet で PostgreSQL を動かす場合の限界を調べる

## 関連ノート

- [[WebSocket の仕組み]] — クラスタ内の Pod 間通信の土台になる HTTP/1.1 と TCP の話

## 出典

- Kubernetes Documentation: https://kubernetes.io/docs/
- Kubernetes API Reference: https://kubernetes.io/docs/reference/kubernetes-api/
- Burns, Beda, Hightower, *Kubernetes Up & Running*
