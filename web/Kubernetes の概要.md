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
- どのコンテナがどこで動いているかを、互いにどう見つけるのか

Kubernetes はこれを宣言的な設定ファイルで扱えるようにする。人間が手順を書くのではなく、最終的にこうなっていてほしいという状態を書き、差分を埋めるのはクラスタの仕事になる。

## 前提となる知識

| 知っておくこと | なぜ必要か |
| --- | --- |
| コンテナとイメージ | Kubernetes が動かす対象。イメージは実行環境のスナップショット、コンテナはその実行 |
| Linux の namespace と cgroups | コンテナの隔離とリソース制限の実体。Kubernetes はその上に乗っているだけ |
| IP・DNS・ポート | Pod 間通信、Service の名前解決、Ingress の振り分けが全部ここ |
| プロセスとシグナル | 停止時に SIGTERM を送って猶予後に SIGKILL を送る。graceful shutdown の設計に効く |
| 宣言的設定と制御ループ | Kubernetes の中心思想。あるべき状態と今の状態を比べて差分を埋める |
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

api-server が唯一の入口になる。`kubectl` もコントローラも kubelet も、全部 api-server と会話する。状態は etcd に保存され、複数ノードで合意を取る。この合意アルゴリズムが Raft（分散システムで合意を取るためのアルゴリズム）になる。

### 2. 宣言的 + 制御ループ（reconciliation）

Kubernetes の本質はここに集まる。

1. あなたが「Pod を3つ維持して」と書く（desired state）
2. etcd に保存される
3. コントローラが定期的に今の状態と比べる
4. 足りなければ作る、多ければ消す

だから落ちても勝手に戻る。誰かが手で消しても、次のループで復活する。命令ではなく状態を書くのがポイントになる。

### 3. Pod が最小単位

コンテナを直接ではなく、Pod という箱に入れて動かす。Pod 内のコンテナはネットワーク（同じ IP、`localhost` で会話できる）、ボリューム、ライフサイクルを共有する。

この単位になっているのは、補助的な処理を同じマシンで同じネットワークに置きたいユースケースが多いからになる。ログ転送、プロキシ、初期化処理などのサイドカー（主処理に付き添う補助コンテナ）がこれにあたる。

Pod は使い捨てで、再起動すると IP も変わる。だから Pod を直接運用せず、上位のオブジェクトに管理させる。

### 4. Deployment → ReplicaSet → Pod

```
Deployment      （このイメージを3つ、こう更新して）
   └ ReplicaSet （3つ維持する）
        └ Pod, Pod, Pod
```

Deployment はローリング更新やロールバックを扱い、ReplicaSet は指定数の Pod を維持し、Pod が実際の実行単位になる。

ローリング更新は、新しい ReplicaSet を少しずつ増やしながら古いものを減らすことで実現する。速度と可用性のバランスは `maxSurge` と `maxUnavailable` で決める。

### 5. Service と Ingress

Pod の IP は変わるので、変わらない入口が必要になる。それが Service になる。

| 種類 | 役割 |
| --- | --- |
| ClusterIP | クラスタ内だけで名前解決できる仮想 IP（既定） |
| NodePort | 各ノードのポートで外から入れる |
| LoadBalancer | クラウドの LB を払い出す |
| ExternalName | DNS のエイリアス |

Service はラベルセレクタで Pod を選ぶ。IP ではなくラベルで結びつくので、Pod が入れ替わっても追従する。内部 DNS では `my-svc.my-namespace.svc.cluster.local` のように引ける。

Ingress は HTTP のルーティングで、パスやホスト名で振り分ける。ただし Ingress リソース自体は設定の宣言にすぎず、実際に動かすには Ingress controller（nginx、Traefik、AWS Load Balancer Controller など）が別途必要になる。

### 6. 設定と秘密

ConfigMap は設定値を環境変数やファイルとして注入する。Secret はパスワードやトークンを扱うが、中身は base64 で符号化されているだけで暗号化ではない。etcd の暗号化や外部の秘密管理（Vault、Sealed Secrets など）と組み合わせて使う。

### 7. スケジューリングとリソース

requests は、この Pod には最低これだけ CPU とメモリを確保してね、というスケジューラへの申告になる。limits はこれ以上使わせない上限で、CPU は throttle され、メモリは OOMKill される。

requests を書かないと、ノードが詰まってから OOM や Evicted で落ちる。最初に書くべきは requests になる。

このほかに、ノード側から排除する taints と tolerations、Pod を寄せたり離したりする affinity がある。

### 8. ストレージとステートフル

Volume は Pod にマウントする記憶領域で、emptyDir は Pod の寿命と同じになる。PersistentVolume と PVC（PersistentVolumeClaim、ストレージの要求）でストレージを抽象化し、StorageClass（ストレージの種類を定義したもの）で動的な確保を指定する。

StatefulSet は、データベースのように名前とディスクが固定の Pod を扱う。ステートフルなものは Kubernetes が苦手な領域で、マネージド DB を使う判断も普通になる。

### 9. ヘルスチェック

liveness は死んでいたら再起動する。readiness は準備できていなければ Service の宛先から外す。startup は起動が遅いアプリ用で、これが通るまで liveness は待つ。

readiness を書かないと、起動途中の Pod にリクエストが飛んで 502 になる。

### 10. よくある誤解

Kubernetes は Docker の代替ではない。コンテナを動かすランタイム（containerd など）の上で、それらを管理する層になる。

Pod は消えるものなので、IP も名前も固定されない。状態を持つなら外に出す。

YAML を書けば動くわけではなく、requests、probe、Service のセレクタが揃って初めて実用的になる。

1台で足りる規模なら過剰になる。Docker Compose で済むなら、そちらのほうが運用は軽い。

## 具体例・コード

最小の Deployment と Service。

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

- `CrashLoopBackOff`。起動して落ちるを繰り返している状態で、`kubectl logs --previous` で前回のログを見る。
- `ImagePullBackOff`。イメージ名の誤り、レジストリの認証、ネットワークの遮断が原因になる。
- Service に繋がらない。セレクタのラベル不一致、`targetPort` の誤り、readiness が通っていないことが多い。`kubectl get endpoints <svc>` が空ならラベルを疑う。
- Pod が Pending になる。リソース不足、taints、PVC がバインドされないことが原因になる。
- OOMKilled。limits を超えた状態で、メモリの実測値を見てから決める。
- `kubectl apply` したのに反映されない。namespace の指定漏れ（`-n`）や、immutable なフィールドの変更が原因になる。
- Ingress を作ったのに 404 になる。Ingress controller が入っていないか、class の指定が漏れている。
- 全部 latest タグにする。どのバージョンが動いているか分からなくなるので、タグは固定する。

## 疑問・TODO

- [ ] kind で1ノードクラスタを立てて、Deployment のローリング更新を観察する
- [ ] readiness を外したとき、更新中にどれくらい 502 が出るか測る
- [ ] HPA（負荷に応じて Pod 数を自動で増減する仕組み）が効く条件を、CPU とレイテンシで比較する
- [ ] eBPF ベースの CNI（Cilium）と kube-proxy の違いを整理する
- [ ] StatefulSet で PostgreSQL を動かす場合の限界を調べる

## 関連ノート

- [[WebSocket の仕組み]] — クラスタ内の Pod 間通信の土台になる HTTP/1.1 と TCP の話

## 出典

- Kubernetes Documentation: https://kubernetes.io/docs/
- Kubernetes API Reference: https://kubernetes.io/docs/reference/kubernetes-api/
- Burns, Beda, Hightower, *Kubernetes Up & Running*
