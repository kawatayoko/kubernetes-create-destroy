# kubenetes
- Control Panel
    - kube-apiserver
        - kubectlコマンド
- Worker Node
    Worker Nodeのことを略してNodeという

# さまざまなKubernetesクラスタ構築環境
- ローカルクラスタ
    - minikube
    - kind
    - k3s
- ベンダー
    - Google Kubernetes Engine（GKE）
    - Amazon Elastic Kubernetes (EKS) 
    - Azure Kubernetes Service (AKS)

## kubernetesクラスタ環境構築
kubectlを使うためにクラスタ環境を構築しないといけない
### kindを利用してクラスタ環境を構築する
事前にdocker desctopを立ち上げておく必要がある。
```
# クラスタ構築
kind create cluster --image=kindest/node:v1.29.0
# kubectlを利用してクラスタと接続できることを確認
kubectl cluster-info --context kind-kind
```
- --contextオプションで利用するクラスタのコンテキストを指定する。
- contextが一つの場合は指定しなくてOK
- contextを切り替えることでクラスタ毎に仕様するconfigの内容を使い分ける

### kindを利用してクラスタ環境を削除
```
kind delete cluster --name kind
```

# pod
kubenetesの最小構成単位
podは複数コンテナをまとめて起動できる

# マニフェストをKubernetesクラスタに適応する
kubectl apply --filename chapter04/myapp.yaml --namespace default
# リソース情報の取得
`kubectl get <リソース名>`
kubectl get pod --namespace default
## outputオプション
kubectl get pod --output wide --namespace default
kubectl get pod --output yaml --namespace default
### outputオプションにjsonpath
kubectl get pod myapp --output jsonpath='{.spec.containers[].image}' --namespace default
kubectl get pod myapp --output json --namespace default | jq '.spec.containers[].image'
## ログレベルを指定
kubectl get pod myapp --v=7 --namespace default 
## リソースの詳細を取得
kubectl describe pod myapp --namespace default
## コンテナログを取得する
kubectl logs myapp --namespace default
## ラベルを指定
kubectl get pod --selector app=myapp
kubectl logs --selector app=myapp
## debug用コンテナを立ち上げる
デバッグ用コンテナを起動することで、多様なデバッグツールを利用できるようになる。
デバッグ用コンテナのイメージは任意のイメージを指定できるので、自分用カスタムコンテナイメージを作っても良い
```
# kubectl debug --stdin --tty <デバッグ対象Pod名> --image=<デバッグ用コンテナのimage> --target=<デバッグ対象のコンテナ名>
kubectl debug --stdin --tty myapp --image=curlimages/curl:8.4.0 --target=hello-serer --namespace default -- sh
```
## コンテナを即座に起動する（kubectl run）
kubectl debug以前は、kubectl runを使ってPodを起動する必要があった
```
# kubectl run <Pod名> --image=<イメージ名>
kubectl --namespace default run busybox --image=busybox:1.36.1 --rm --stdin --tty --restart=Never --command -- nslookup google.com
```
--rm : 実行が完了したらPodを削除
--stdin(i) : オプションで標準入力に渡す
--tty(t): オプションで擬似端末を割り当てる
--restart=Never デフォルトでは常に再起動するポリシーのため、一回きりのコマンドを実行する場合、指定する必要がある
--command --: "--" の後に渡される文字列をコマンドとして扱う

## コンテナにログイン
```
# kubectl exec --stdin --tty <Pod名> -- <コマンド名>
kubectl --namespace default run curlpod --image=curlimages/curl:8.4.0 --command -- /bin/sh -c "while true; do sleep infinity; done;"
kubectl get pod --namespace default
# IPアドレスを確認
kubectl get pod myapp --output wide --namespace default
# curlpodに入ってcurlを実行
kubectl --namespace default exec --stdin --tty curlpod -- /bin
/sh
~ $ curl 10.244.0.5:8080

```

## port-forwardでアプリケーションにアクセス
```
# kubectl port-forward <Pod名> <転送先ポート番号>:<転送元ポート番号>
kubectl port-forward myapp 5555:8080 --namespace default
# 確認(別ターミナルで)
curl localhost:5555
```

# 障害対応のためのkubectl コマンド
## マニフェストをその場で編集
```
# kubectl edit <リソース名>
kubectl edit pod myapp --namespace default
```
## リソースを削除する
```
# kubectl delete <リソース名>
# kubectlにはPodを再起動するというコマンドがないため代替される

# pod確認
kubectl get pod --namespace default
# pod削除
kubectl delete pod myapp --namespace default
```
## kubectrlのチートシート
https://kubernetes.io/docs/reference/kubectl/quick-reference/

## kubectrl役立つツール
- k9s  
    https://k9scli.io/topics/commands/
- starship  
    https://starship.rs/
- kubectx/kubens

# Chapter　6　Kubernetesリソースを作って壊そう
## 6.2 Podを冗長化するためのReplicaSetとDeployment
- 実際の運用現場ではPodを直接作ることは推奨されていない
    - Pod単体ではコンテナの冗長化ができないので
- DeploymentとReplicaSet
    - ReplicaSetは複数Podをまとめたもの
    - Deploymentは複数ReplicaSetをまとめたもの
    ```mermaid
    graph LR

    RS1[ReplicaSet] --> P1[Pod]
    RS1 --> P2[Pod]

    D[Deployment]

    D -->|v2| RS2[ReplicaSet]
    D -->|v1| RS3[ReplicaSet]

    RS2 --> P3[Pod]
    RS2 --> P4[Pod]

    RS3 --> P5[Pod]
    RS3 --> P6[Pod]
    RS3 --> P7[Pod]
    ```
### ReplicaSet
- 指定した数のPodを複製するリソース
- 複製するPodの数をreplicasで指定する
- ReplicaSet作成のコマンド
```sh
# Replica作成
kubectl apply --filename chapter-06/replicaset.yaml --namespace default

# Pod確認
kubectl get pod --namespace default

# ReplicaSetのリソース確認
kubectl get replicaset --namespace default

# ReplicaSetのリソース削除
kubectl delete replicaset --namespace default
```
### Deployment
- 本番の運用環境ではPod更新時に「無停止で更新する」ことが必要
- Deployementは、ReplicaSetを複数管理し、ローリングアップデートを実現できる

```sh
# deploymentを作成
$ kubectl apply --filename chapter-06/deployment.yaml --namespace default
deployment.apps/nginx-deployment created

# deploymentの作成を確認
$ kubectl get deployment --namespace default
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           84s

# Podが作成されていることを確認
$ kubectl get pod --namespace
 default

# ReplicaSetが作成されていることを確認
$ kubectl get replicaset --namespace default

# 作成したマニフェストを参照
# StrategyType, RollingUpdateStrategy を参照できる
kubectl describe deployment nginx-deployment
```
- StrategyType
    - どのような先約でPodを更新するか？を定義
    - Recreate
        - 全部のPodを同時に更新
    - RollingUpdate
        - Podを順番に更新
        - RollingUpdateStrategyを記載できる

### 6.2.3 作って直す　Deploymentをつくって壊そう
- 親子関係の整理
    - Deployment > ReplicaSet > Pod > Container
- RollingUpdate時の設定項目
    - maxUnavailable: 更新中に利用不能になってよいPodの最大数。小数点以下は切り下げ
    - maxSurge: 更新中に一時的に増やしてよいPodの最大数。小数点以下は切り上げ
    - Replicas: Pod数

# 6.3 Podへのアクセスを助けるService
- Service
    - Deploymentで作成した複数Podへのアクセスを適切にルーティングするために利用するリソース
    - Type
        - ClusterIP: 
            クラスタ内部のIPアドレスでServiceを公開    
            ClusterIPはクラスタ内部からしか疎通できない。
            外部公開するにはIngressというリソースを利用する
            -> クラスタ内部とは？
        - NodePort:
            全ておNodeのIPアドレスで指定したポート番号を公開する
            クラスタ外からもアクセス可能。port-fowardする必要がなくなる。
            Nodeが故障すると使えなくなるので、本番環境ではClusterIPやLoadBalancerを利用するほうがよい
        - LoadBalancer:
            外部ロードバランサを用いて外部IPアドレスを公開
            LBは別途よういする必要あり
        - ExternaName:
            ServiceをexternalNameフィールドの内容にマッピングする
            （たとえば、ホスト名api.example.com）
    - Serviceを利用したDNS
        - KubernetesではService用のDNSレコードを自動で作成してくれる
            <Service名>.<Namespace名>.svc.cluster.local
            で接続可能
            ```
            kubectl apply --filename chapter-06/service.y
aml --namespace default
            kubectl apply --filename chapter-06/deploymen
t-hello-server.yaml --namespace default
            kubectl --namespace default run curl --image curlimages/curl --rm --stdin --tty --restart=Never --command -- curl hello-server-service.default.svc.cluster.local:8080
            ```
# 6.4 Podの外部から情報を読み込むConfigMap
- ConfigMap
    - コンテナの外部から値を設定したいときに利用するリソース
- 利用方法(2と３がよく使われる)
    1. コンテナ内のコマンドの引数として読み込む
    2. コンテナの環境変数として読み込む
        - 変更後アプリケーションの再作成が必要
        ```
        kubectl rollout restart deployment/hello-server --namespace d
efault
        ```        
    3. ボリュームを利用してアプリケーションのファイルとして読み込む
        - アプリケーションの再作成不要
        - ボリューム
            Pod間でファイルを共有できるファイルシステム
        
- コンテナの環境変数として読み込む

# 6.5 機密データを扱うためのSecret
- Secretリソース
    - Base64でエンコードして登録

# 6.6 1回限りのタスクを実行するためのJob
- kind: Job
# 6.7 Jobをテウイキ的に実行するためのCronJob
- CronJobリソース
    - 定期的にJobを生成するリソース
    - CronJobはJobを作成、JobはPodを作成する

# 7.1 アプリケーションのヘルスチェックを行う
- Readiness probe
    - コンテナがReadyになるまでの時間やエンドポイントを制御する
    - 失敗したらServiceから接続をはずす
- Liveness probe
    - Readminess probeとの違いは失敗時の挙動
    - 失敗時はPodを再起動する
        - 再起動を無限に繰り返してしじゃうリスクがあるため安易に導入しない
        - 再起動でなおるケースが想定される場合に有効
    - Readiness probeとの同時設定は可能
        - Readiness probeが先に実行されることが推奨される
            - initialDelaySeconds で調整する
- Startup probe
    - Pod初回起動時のみに使用するProbe

# 7.2 アプリケーションに適切なリソースを指定しよう
CPU・メモリ・EphemeralStrage　が指定できる
- Resource requests
    - コンテナのリソース使用量（下限）を要求する
    - kubernetesのスケジューラはこの値を参照してスケジュールするNodeを決定する
        - Nodeは物理サーバー、EC2、仮想マシンなどのPodをうごかすマシンのこと
        - KubernetesはPodをどのNodeで動かすかを自動で決める
            - その決定をするのがscheduler
    - コンテナごとにCPUとメモリを指定できる
- Resource limits
    - コンテナのリソース使用量（上限）を指定する
    - CPUが上限を超えた場合、スロットリングが発生する
    - メモリが上限を超えた場合、Out Of Memory(OOM) でPodはkillされる
- メモリ
    - 単位を指定しない場合、1 = 1byte
    - K, M, Ki, Mi などを指定可
- CPU
    - 単位を指定しない場合、1 = 1コア
        - 1m = 0.001 コア
        - 通常、整数 or ミリコアで指定する
- PodのQuality of Service (QoS) Classes
    - Nodeのメモリが完全に枯渇すると、Nodeに載っている全てのコンテナが動作できなくなる
        - それを防止するためOOM Killerというプログラムがある
        - OOMKillerは、OOMkillするPodの優先順位を決定し、優先度の低いPodからOOMkillする
    - 優先順は以下
        1. Guaranteed
            Pod内のコンテナすべてにリソースのrequestsとlimitsが指定されている
            メモリとCPUの値が、requests = limitsとなっている 
        2. Burstable
            Pod内のコンテナのうち、すくなくとも一つはメモリorCPUのrequests/limitsが指定あれている
        3. BestEffort
            Guaranteed, Burstableでないもの
    - 以下のコマンドでQoSクラスを確認できる
        `kubectl get pod hello-server --output jsonpath='{.status.qosClass}' --namespace default`

# 7.3 Podのスケジュールに便利な機能
NodeとPodの関係性の制御
- Node selector
    - 特定のNodeにのみスケジュールする
        SSDをつかっているNodeにのみ`disktype: ssd`というラベルが付与されている場合
        `nodeSelector: ssd`とマニフェストで指定することで、SSDを使っているNodeにのみPodをスケジュールできる
- Affinity/Anti-affinity
    - Affinity：類似性、密接な関係
    - NodeとPod、あるいはPod同士が「近くなるように」「近づかないように」スケジュールする
    - 3種類ある
        - Node affinity
            - Node selectorと近い。Node selectorより柔軟に設定できる。
            - 「可能ならスケジュールする」という選択ができる
            - `requiredDuringSchedulingIgnoredDuringExecution`
                - 対応するNodeが見つからない場合、Podをスケジュールしない
                - Node selectorと同じ考え方だが、Nodeの指定方法より柔軟
            - `preferredDuringSchedulingIgnoredDuringExecution`
                - 対応するNodeが見つからない場合、適当なNodeにスケジュールする
            - `matchExpressions`を利用してNodeを指定する
        - Pod affinity / Pod anti-affinity
            - spec.affinity以下にpodAddinity/podAntiAffinityを指定する
            - Pod間のAffinity
            - すでにNodeにスケジュールされている既存Podのラベルに基づいて新規Podのスケジューリングする
            - ユースケース
                - Node障害に備えて「同じアプリケーションを動かしているPodは同じNodeに乗せない」
            - Node affinityと同様に以下のルールを設定可能
                - `requiredDuringSchedulingIgnoredDuringExecution`
                - `preferredDuringSchedulingIgnoredDuringExecution`
        - Pod Topology Spread Constraints
            - Podを分散するための設定
            - topologyKeyを使うことでどのようにPodを分散させるかを表現できる
            - hostnameラベルを指定するとホスト間でPodを分散してスケジュールできる
            - Pod anti-affinitiyでもおなじような設定ができるが、より柔軟に設定可能
            - Pod Topology Spread Constraints ではPod数がNode数を超えてもなるべく分散させる設定ができる
        - Taint / Toleration
            - Taint: Nodeに付与する設定
                - 「汚れ」の意味
            - Toleration: Podに付与する設定
                - 「寛容」の意味
            - Traint（汚れ）をPodが許容できるか？という考えかたの設定
            - あるNodeが特定のPodしかスケジュールしたくない、という指定方法
            `kubectl taint nodes <対象ノード名> <label名>=<labelの値>:<Taintの効果>`
        - Pod PriorityとPreemption
            - Priority
                - PodにPriorityを設定できる
                - PriorityClassというリソースを使って設定する
                    1. PriorityClassを作成する
                    2. 1で設定したPriorityClassをPodのマニフェストに指定する
            - Preemption
                - priorityが低いPodをEvicctさせること
# 7.4 アプリケーションをスケールさせよう
- 水平スケール（Horizontal Pod Autoscaler: HPA）
    - サーバーの台数を増やす
    - Pod数を増やす・減らす
    - HPAを利用するにはmetrics-serverのインストールが必要
    - インストール
    ```
    kubectl apply --filename https://github.com/kubernetes-sigs/metric
s-server/releases/download/v0.6.4/components.yaml
    kubectl patch --namespace kube-system deployment metrics-server --type=json --patch '[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'
    ```
- 垂直スケール(Vertical Pod Autoscaler: VPA)
    - サーバーの使用リソースを増やす
    - VPAを利用することで自動でResource Requests/Limits の値を変更できる
    - HPAと同じリソースに同時に適応不可
    - なのでHPAのみ利用しているケースがおおい
    



#　chapter5以降は以下コマンドで二つpodを立ち上げる

## 事前準備
### 準備１） 以下コマンドを実行するにはDocker Desktopを起動しておくこと。
### クラスターの状態確認
kind get clusters
## クラスタ起動
kubectl apply --filename chapter-04/myapp.yaml
kubectl run myapp2 --image=blux2/hello-server:1.0 --namespace default
kubectl apply --filename chapter-05/myapp-label.yaml